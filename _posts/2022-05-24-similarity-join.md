---
title: Similarity join (Min-hash)
layout: post
---


Motivation
----------

There are lot of use cases where we might like to join two datasets not based on exact values, but on how similar they are.  There is whole suite of applications for this, like entity resolution, de-duplication, plagiarism detection etc.. We will go over a very performant and extensible architecture to do joins based on similarity using a technique called Min-hashing. We will extend the Min-hashing idea some more by introducing weights/boost to fields of the document and particular terms/tokens based on their IDF values.

We will consider documents/records to be set of words/tokens without repetitions for the sake of simplicity, and we won't deal with finer points of tokenization, phrase detection or shingles also in this article.  

Coming back to the problem, consider two spark's RDD (`RDD[(Int, String)]`)  where `Int` refers to the identifier of a single document and the `String` value is the actual document. The straight-forward brute force approach to do similarity based joins, would be to emit all tokens and then aggregate and/or filter them by how many of the tokens each pair of records have in common. Something like this

{% highlight scala %}
def tokenize(string: String): List[String] = ...
val left: RDD[(Int, String)] = ... // Int is the id of the document and String is the document
val right: RDD[(Int, String)] = ...

def tokenizeDocs(rdd: RDD[(Int, String)]) = rdd.flatMap { case (id, doc) =>
  tokenize(doc).map(_ -> id)
}

tokenizeDocs(left)
  .join(tokenizeDocs(right)) // Join by token
  .map(_._2 -> 1) // Take just pairs of docs
  .reduceByKey(_ + _) // Each pair of documents how many tokens match
{% endhighlight %}

Jaccard's similarity
----------

We found the number of tokens that match between two documents, but we would really like a normalized score (between 0 to 1) representing how similiar two documents are, so that we can make compare matches and can also have some sort of threshold filtering. There are several ways to score how similar a pair of records are. One simple metric is called Jaccard's similarity. 

Suppose we consider each document to be a set of tokens, let's call it $A$ and $B$ . Then, the Jaccard's similarity between $A$ and $B$ is defined to be,

$$\frac{| A \cap B|}{| A \cup B|}$$

Basically, number of tokens in common divided overall number of tokens. We can tweak the above scala code to also carry over number of records on left RDD and right RDD, then we would be able to calculate the Jaccard's similarity as

$$\frac{|A \cap B|}{|A|+|B| - |A \cap B|}$$

Random choice and Jaccard's similarity
----------

The above scala source has one other drawback, it has to emit a ton of tokens depending on the size of the documents, Also if the documents all talk about the same domain, then they are all bound to have atleast one token matching, which quadratically explodes the number of pairs. Suppose if we have 1-million records on each RDD (left and right) then the number of combinations we need to calculate the similarity on is about 1-trillion!. This could go quickly out of hand moment we have more and more records. It would be nice if could join only those pairs of records which has over particular Jaccard's similarity score.

There is a probabilistic approach which can do just that. Consider the scenario, $$A = \{a,b, c\}$$ and $$B = \{c, d\}$$ . The Jaccard's similarity score is $0.25$. Now, if we randomly chose an element each from $A \cup B$ , the probability that the element is in both $A$ and $B$ is equal to the Jaccard's similarity! We can exploit this to do the similarity based join.

Shuffling and picking first row
----------

Let's hypothetically build a characteristic matrix (true/false based on whether the item is present on not, just like how `Set[T]` implements `T => Boolean`) of all the tokens in all our documents as below,

| Token |  A     | B     |  C     |
| ----- | ----- | ----- | ----- |
| a     |  true  |  false |  true  |
| b     |  false |  true  |  false |
| c     |  false |  false |  true  |
| d     |  true  |  false |  false |

The above table represents the following documents $$A = \{a,d\}, B = \{b\}, C=\{a,c\}$$

Suppose we shuffle the rows in the above table and find the first row which has $true$ in the $A$'s column.  The probability of $C$ also having $true$ in the row would be equal to Jaccard's similarity! Don't move on until this is clear.

But it's quite in-feasible to build such a sparse matrix let alone shuffle one. We are going to simulate this with a  individual process per row, in such a way that it looks like we created and shuffled above characteristic table without building one,  so that we can parallelize it.

Min hashing
----------

In order to simulate the shuffling, we should choose a random hash function `String => Int`, we will discuss later how to build one. We will apply the chosen hash function to every token in a given document and choose the element with the smallest value. That forms the representative for the document. The probability of any token being chosen is equal to $1/n$ where $n$ is the number of tokens in the document, provided the hash function is random. This simulates the shuffling and picking the first row which contains $true$ in a given column in the above table. We do the same for every document. Thus every element have received a particular token. 

The probability that two documents $A$ and $B$ having the same representative token is, equal again to Jaccard's similarity This process is called **min-hashing**.

Rows and bands
----------

It's necessary to repeat the above process considerable number of times with different but fixed hash functions so as to get a better coverage. Usually we take $r \times b$ hash-functions for some $r, b > 0$ . We take $b$ bands of $r$ hash values each. For example, if we choose $b = 6, r = 3$ then we take $18$ hash function $\{h_1, h_2, ... h_{18}\}$ where each $h_i$` : String => Int`, the bands would be like $\{(h_1, h_2, h_3), ... (h_{16}, h_{17},h_{18})\}$. We emit (`flatMap`) each band with the record and join the left and right RDD using emitted band. Using $b$ bands of $r$ hash values as the join key has some really nice properties.

Suppose $s$ be the similarity score between document $A, B$ then

- Probability of a single hash in a single band (ie.,  a single hash value) matching is equal to $s$.
- Probability of a particular band matching (ie., $r$ hash values) is equal to $s^r$
- Probability a particular band not matching (ie., $r$ hash values)  is equal to $1 - s^r$
- Ptobability none of the bands (ie,, $b$ bands of $r$ hash values each) not matching is equal to $(1 - s^r)^b$
- Probability that atleast one of the bands matching is equal to $1 - (1 - s^r)^b$

The plot of function $f(s) = 1 - (1 - s^r)^b$ for fixed $r, b$ looks like this. 

![fixed-row-band](/assets/images/minhash/fixed-row-band.gif)

If we increase $r$ keeping $b$ fixed the graph translates to the left, which mean pairs having lower similarity are getting *filtered out* more often.



![increase-row-band](/assets/images/minhash/increase-row-band.gif)

Suppose we increase $b$ keeping $r$ fixed the graph gets steeper, which mean more pairs above the threshold gets *filtered in* more often.

![row-band-curve](/assets/images/minhash/row-increase-band-curve.gif)

(Play around with different values of $r$ and $b$ [here](https://www.wolframalpha.com/input?i=plot+%281+-+%281+-+s%5Er%29%5E+b%29+where+0+%3C%3D+s+%3C%3D+1%2C+r+%3D+4%2C+b+%3D+10) )

 In some sense the $r$ and $b$ values act as knobs to control the precision and recall (and inversely performance) of the joins. 

Practical hash functions
----------

How do generate a random hash function let alone $r \times b$  one of them? 

All java (and hence scala) `String` objects have quite good hash function to hash a `String` to an `Int` in the form of `hashCode` function. We use the `hashCode` to generate more hash functions by generating $r \times b$ random integers and `xor` ing each of them with the `hashCode`.

The consolidated algorithm to do similarity based join, looks like this

{% highlight scala %}
import scala.util.Random

val rows: Int = ???
val bands: Int = ???

val hashFns = 0.to(rows * bands).map(_ => Random.nextInt()).toArray

def minHash(value: String): Array[Array[Int]] = {
  val tokens = tokenize(value).map(_.hashCode).toSet
  def doMinHash(hashFn: Int) = {
    tokens
    .map(_ ^ hashFn)
    .min
  }
  
  hashFns
    .map(doMinHash(_))
    .grouped(rows)
}

def emitBands(rdd: RDD[(Int, String)]) = {
  rdd.flatMap { case (id, document) =>
    minHash(document).map { band => (band, id) }
  }
}

emitBands(left)
  .join(emitBands(right))
  .map { case (band, (left, right)) => (left, right) } 
// We can join back to the original document to do some more processing after this.
{% endhighlight %}

Integer weights
----------

Lot of times, we will have documents with several fields, some more important than other. And we would like to boost matches in important fields over the other based on some empirical evidence or just plain intuition. Consider the hypothetical charateristic matrix, if we repeat tokens of certain fields more often than the tokens of other fields, we are in effect increasing the probability of them being chosen over the other when we shuffle and pick, and hence boost their contribution to the similarity score. 

However, repeating a token with hashing has no effect, as the repeated values all have the same `hashCode` value and therefore doesn't have increased chance of being picked up as minimum. The trick is to have another set of random integers fixed before hand, as many as the weight, which should be used to hash the tokens of a particular filed again and thereby giving more chance for the hash value to be smaller and be minimum. It's essential to have same boost functions and weight on both left and right side to make this work.

Like below.,

{% highlight scala %}
case class Field[T](accessor: T => String, boost: Int)

val maxBoost: Int = ???
val hashFns = 0.to(rows * bands).map(_ => Random.nextInt()).toArray
val boostFns = 0.to(maxBoost).map(_ => Random.nextInt()).toArray

def minHash[T](value: T, fields: List[Field[T]]): Array[Array[Int]] = {
  val fieldHashCodes = fields.map { field => 
  	val tokens = tokenize(field.accessor(value))
      .map(_.hashCode)
      .toSet
    
    (tokens, field.boost)
  }
  
  hashFns.map { hashFn =>
    fieldHashCodes.map { case (tokens, boost) =>
      /* We take the minimum first before applying boost to circumvent undue influence of a field with lot of tokens and lower boost value to trump over the field with smaller number of tokens */
      val minTokenCode = tokens.map(_ ^ hashFn).min 
      boostFns.take(boost).map(minTokenCode ^ _).min
    }.min
  }.grouped(rows)
}
{% endhighlight %}

IDF values
----------

Not all tokens of a document should be treated equal, as in some of the tokens are just repeated too often that they shoul contribute lesser to the over all similarity where-as a token which is much rarer should contribute more. As in we need to boost per token in a field based on the information retrieval metric called IDF or **inverse document frequency**

The inverse document frequencies are usually calculated as 

$$log(\frac{N}{N_t})$$

Where $N$ is the total number of documents and $N_t$ is the number of documents containing the token $t$ (Do note, that we don't consider repetition of a token in a single document, which can be fixed quite easily). This value increases with the rarity of a token and decreases with how frequent a token appears in the document. One of the reasons (intuitively) for using logarithms instead of just fractions is, the frequency increase from 1 to 2 matters more than say 100 to 101. 

If IDF value is rewritten like this using logarithm's identity,

$$log(N) - log(N_t)$$

And we take `ceil` (or `floor`), we get an integer value which could be used as proxy for IDF. Note that, $log(N)$ will be the maximal value we can get.

Count-Min Sketch for document frequency calculation
--------

The most straight-forward way to calculate the IDF values are to run another spark job to collect the number of repetition of each token and then consume that while calculating `minHash` . But this join is to going to slow down the spark job a lot, if only we could do a map-side join (aka., broadcast joins). The broadcast joins sends the entire RDD across to all the partitions of another RDD if the broadcasted RDD is small enough, so there is no exchange on the larger RDD. However the size of both the RDD having documents and RDD having token frequencies are large, and hence can't be directly broadcasted.

Notice that, we don't really need very precise count of frequency, an approximate count is all we need, since we lose a lot of precision when taking logarithms and bucket them into integers. So a  [sketch](https://en.wikipedia.org/wiki/Category:Probabilistic_data_structures) datastruture, which gives approximate count would do. Count-Min sketch does exactly that, and an implementation of it comes with spark libraries.

Count-Min Sketch could be thought of as `Map[String, Int]` which maps each string to how many times it occured (approximate, lower estimate) in sub-linear space. Once we have the approximate count, we could have another set of `hashFns` which would boost a particular token as many times as the bucketed idf value dictates.

{% highlight scala %}
import org.apache.spark.util.sketch._

case class Field[T](name: String, accessor: T => String, boost: Int)

val left: RDD[T] = ???
val right: RDD[T] = ???

def frequencies[T](rdd: RDD[T], fields: List[Field[T]]): Map[String, CountMinSketch]= {
  rdd.flatMap { record => 
  	fields.flatMap { field => 
    	val tokens = tokenize(field.accessor(record)).toSet
      tokens.map(token => (field.name, token) -> 1l)
    }
  }
  .reduceByKey(_ + _)
  .map { case ((name, token), count) => name -> (token, count)}
  .combineByKey(
  	_ => CountMinSketch.create(0.000001, 0.99, 678),
  {(sketch: CountMinSketch, value: (String, Long)) => sketch.addString(value._1, value._2); sketch},
  (left: CountMinSketch, right: CountMinSketch) => left.mergeInPlace(right)
  )
  .collectAsMap 
}

def total[T](rdd: RDD[T]): Long = rdd.count

val docFrequencies = frequencies(left.union(right))
val N = total(left.union(right))


val maxBoost: Int = ???
val hashFns = 0.to(rows * bands).map(_ => Random.nextInt()).toArray
val boostFns = 0.to(maxBoost).map(_ => Random.nextInt()).toArray
val idfBoostFns = 0.to(math.ceil(math.log(N)).toInt + 1).map(_ => Random.nextInt()).toArray

def proxyIdf(field: String, token: String) = {
  val Nt = (docFrequencies(field).estimateCount(token) + 1) // To smooth out divide by zero
  math.ceil(math.log(N/Nt))
}

def minHash[T](value: T, fields: List[Field[T]]): Array[Array[Int]] = {
  val fieldHashCodes = fields.map { field => 
    val tokens = tokenize(field.accessor(value))
      .map(token => (token.hashCode, proxyIdf(tokens))
      .toSet

    (tokens, field.boost)
  }
  
  hashFns.map { hashFn =>
    fieldHashCodes.map { case (tokens, boost) =>
      val minTokenCode = tokens.map{ case (hashCode, idf) =>
        val firstLevel = hashCode ^ hashFn
        // Hash it atleast once again
        idfBoostFns.take(idf + 1).map(_ ^ firstLevel).min
      }
      boostFns.take(boost).map(minTokenCode ^ _).min
    }.min
  }.grouped(rows)
}
{% endhighlight %}

Asymmetric joins
----------

Sometimes we might be doing this similarity joins asymmetrically, as in, one of the RDD is much larger than the other. Hence there is lot of un-necessary, shuffles on the larger side which doesn't match with any of the bands on the smaller side. If we take the count-min sketch as inspiration, we could build bloom filter on either side and filter the bands before they are joined. This would reduce the cost of shuffles. We would only be shuffling bands/records which has considerable chance of finding a match.

{% highlight scala %}
def asymmetricJoin[T, K](left: RDD[(Array[Int], T)], left: RDD[(Array[Int], K)])  = {
  def buildFilter(rdd: RDD[(Array[Int], T)]): BloomFilter = 
    rdd
      .map { case (key, _) => key.mkString(":")}
      .aggregate(BloomFilter.create(50000000l, 0.85))( // Approximate number of records, and confidence
        {(bloom: BloomFilter, record: String) => bloom.putString(record); bloom}, 
        _ mergeInPlace _
      ) 

  def doFilter(rdd: RDD[(Array[Int], T)], filter: BloomFilter): RDD[(Array[Int], T)] = 
    rdd
    .filter{ case (key, _) => filter.mightContainString(key.mkString(":"))}

  val leftFilter = buildFilter(left)
  val rightFilter = buildFilter(right)

  doFilter(left, rightFilter)
    .join(doFilter(right, leftFilter))
  
}
{% endhighlight %}

Conclusion
----------

Do note that we have used RDD api everywhere, though there are some places, like the asymmetric join case where projection of just the keys in dataframe apis could make things faster. But RDD's aggregation api is much nicer and easier to use while calculating sketches hence sticked to it.

There are some more improvement we could do here, like taking repetitions of tokens into account, inferring the weight of fields with some labelled data etc.. But we need to end this article somewhere :)