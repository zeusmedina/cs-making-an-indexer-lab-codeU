# cs-making-an-indexer-lab

## Learning goals

1.  Understand the design of a Web indexer and fill in missing methods.
2.  Choose appropriate data structures from the Java Collections Framework (JCF).
3.  Use Java's `Map` interface and the `HashMap` implementation.
4.  Use the `Set` interface and the `HashSet` implementation.


## Overview

At this point we have built a basic Web crawler; the next piece we will work on is the **index**.  In the context of Web search, an index is a data structure that makes it possible to look up a search term and find the pages where that term appears.  In addition, we would like to know how many times the search term appears on each page, which will help identify the pages most relevant to the term.

For example, if a user submits the search terms "Java" and "programming", we would look up both search terms and get two sets of pages.  Pages with the word "Java" would include pages about the island of Java, the nickname for coffee, and the programming language.  Pages with the word "programming" would include pages about different programming languages, as well as other uses of the word.  By selecting pages with both terms, we hope to eliminate irrelevant pages and find the ones about Java programming.

Now that we understand what the index is and what operations it performs, we can design a data structure to represent it.


## Data structure selection

The fundamental operation of the index is a **lookup**; specifically, we need the ability to look up a term and find all pages that contain it.  The simplest implementation would be a collection of pages.  Given a search term, we could iterate through the contents of the pages and select the ones that contain the search term.  But the runtime would be directly proportional to the total number of words on all the pages, which would be way too slow.

A better alternative is a **map**, which is a data structure that represents a collection of **key-value pairs** and provides a fast way to look up a **key** and find the corresponding **value**.  For example, the first map we'll construct is a `TermCounter`, which maps from each search term to the number of times it appears in a page.  The keys are the search terms and the values are the counts (also called "frequencies").

Java provides an interface called `Map` that specifies the methods a map should provide; the most important are:

* `get(key)`:  This method looks up a key and returns the corresponding value.

* `put(key, value)`:  This method adds a new key-value pair to the `Map`; or if the key is already in the map, it replaces the value associated with `key`.

Java provides several implementations of `Map`, including the two we will focus on, `HashMap` and `TreeMap`.  In upcoming labs, we'll look at these implementations and analyze their performance.

In addition to the `TermCounter`, which maps from search terms to counts, we will define a class called `Index`, which maps from a search term to a collection of pages where it appears.  And that raises the next question, which is how to represent a collection of pages.  Again, if we think about the operations we want to perform, that guides our decision.

In this case, we'll need to combine two or more collections and find the pages that appear in all of them.  You might recognize this operation as **set intersection**: the intersection of two sets is the set of elements that appear in both.

As you might expect by now, Java provides a `Set` interface that defines the operations a set should perform.  It doesn't actually provide set intersection, but it provides methods that make it possible to implement intersection and other set operations efficiently.  The core `Set` methods are:

* `add(element)`:  This method adds an element to a set; if the element is already in the set, it has no effect.

* `contains(element)`:  This method checks whether the given element is in the set.

Again, Java provides several implementation of `Set`; the ones we will focus on are `HashSet` and `TreeSet`.  We'll come back to them later.

Now that we've designed our data structures from the top down, we'll implement them from the inside out, starting with `TermCounter`.


## TermCounter

`TermCounter` is a class that represents a mapping from search terms to the number of times they appear in a page.  Here is the first part of the class definition:

```java
public class TermCounter {

	private Map<String, Integer> map;
	private String label;

	public TermCounter(String label) {
		this.label = label;
		this.map = new HashMap<String, Integer>();
	}
}
```

The instance variables are `map`, which contains the mapping from terms to counts, and `label`, which identifies the document the terms came from; we'll use it to store URLs.

To implement the mapping, I chose `HashMap`, which is the most commonly-used `Map`.  Coming up in a few lessons, you will see how it works and why it is a common choice.

`TermCounter` provides `put` and `get`, which are defined like this:

```java
	public void put(String term, int count) {
		map.put(term, count);
	}

	public Integer get(String term) {
		Integer count = map.get(term);
		return count == null ? 0 : count;
	}
```

`put` is just a **wrapper method**; when you call `put` on a `TermCounter`, it calls `put` on the embedded map.

On the other hand, `get` actually does some work.  When you call `get` on a `TermCounter`, it calls `get` on the map, and then it checks the result.  If the term does not appear in the map, `TermCount.get` returns 0.  Defining `get` this way makes it easier to write `incrementTermCount`, which takes a term and increases by one the counter associated with that term.

```java
	public void incrementTermCount(String term) {
		put(term, get(term) + 1);
	}
```

If the term has not been seen before, `get` returns 0; we add 1, then use `put` to add a new key-value pair to the map.  If the term is already in the map, we get the old count, add 1, and then store the new count, which replaces the old value.

In addition, `TermCounter` provides these other methods to help with indexing Web pages:

```java
	public void processElements(Elements paragraphs) {
		for (Node node: paragraphs) {
			processTree(node);
		}
	}

	public void processTree(Node root) {
		for (Node node: new WikiNodeIterable(root)) {
			if (node instanceof TextNode) {
				processText(((TextNode) node).text());
			}
		}
	}

	public void processText(String text) {
		String[] array = text.replaceAll("\\pP", " ").toLowerCase().split("\\s+");

		for (int i=0; i<array.length; i++) {
			String term = array[i];
			incrementTermCount(term);
		}
	}
```

*  `processElements` takes an `Elements` object, which is a collection of jsoup `Element` objects.  It iterates through the collection and calls `processTree` on each.

*  `processTree` takes a jsoup `Node` that represents the root of a DOM tree.  It iterates through the tree to find the nodes that contain text; then it extracts the text and passes it to `processText`.

*  `processText` takes a String that contains words, spaces, punctuation, etc.  It removes punctuation characters by replacing them with spaces, converts the remaining letters to lowercase, then splits the text into words.  Then it loops through the words it found and calls `incrementTermCount` on each.

Finally, here's an example that demonstrates how `TermCounter` is used:

```java
    String url = "https://en.wikipedia.org/wiki/Java_(programming_language)";
    WikiFetcher wf = new WikiFetcher();
    Elements paragraphs = wf.fetchWikipedia(url);

    TermCounter counter = new TermCounter(url);
    counter.processElements(paragraphs);
    counter.printCounts();
```

This example uses a `WikiFetcher` to download a page from Wikipedia and parse the main text.  Then it creates a `TermCounter` and uses it to count the words in the page.

In the next section, you'll have a chance to run this code and test your understanding by filling in a missing method.


## Finishing off `TermCounter`

When you check out the repository for this lab, you should find a file structure similar to what you saw in previous labs.  The top level directory contains `CONTRIBUTING.md`, `LICENSE.md`, `README.md`, and the directory with the code for this lab, `javacs-lab06`.

In the subdirectory `javacs-lab06/src/com/flatironschool/javacs` you'll find the source files for this lab:

    *  `TermCounter.java` contains the code from the previous section.

    *  `Index.java` contains the class definition for the next part of this lab.

    *  `WikiFetcher.java` contains the class we used in the previous lab to download and parse Web pages.

    *  `WikiNodeIterable.java` contains the class we used to traverse the nodes in a DOM tree.

Also, in `javacs-lab06`, you'll find the Ant build file `build.xml`.

*  In `javacs-lab06`, run `ant build` to compile the source files.  Then run `ant TermCounter`, it should run the code from the previous section and print a list of terms and their counts.  The output should look something like this

        genericservlet, 2
        configurations, 1
        claimed, 1
        servletresponse, 2
        occur, 2
        Total of all counts = -1

When you run it, the order of the terms might be different.

*  The last line is supposed to print the total of the term counts, but it returns `-1` because the method `size` is incomplete.  Fill in this method and run `ant TermCounter` again.  The result should be `4798`.

Run `ant test1` to confirm that this part of the lab is complete and correct.


## Finishing off `Index`

For the second part of the lab, we'll present our implementation of an `Index` object and you will fill in a missing method.  Here's the beginning of the class definition:

```java
public class Index {

	private Map<String, Set<TermCounter>> index = new HashMap<String, Set<TermCounter>>();

	public void add(String term, TermCounter tc) {
		Set<TermCounter> set = get(term);

		// if we're seeing a term for the first time, make a new Set
		if (set == null) {
			set = new HashSet<TermCounter>();
			index.put(term, set);
		}
		// otherwise we can modify an existing Set
		set.add(tc);
	}

	public Set<TermCounter> get(String term) {
		return index.get(term);
	}
```

The instance variable, `index`, is a map from each search term to a set of `TermCounter` objects.  Each `TermCounter` represents a page where the search term appears.

The `add` method adds a new `TermCounter` to the set associated with a term.  When we index a term that has not appeared before, we have to create a new set.  Otherwise we can just add a new element to an existing set.  In that case, `set.add` modifies a set that lives inside `index`, but doesn't modify `index` itself.  The only time we modify `index` is when we add a new term.

Finally, the `get` method takes a search term and returns the corresponding set of `TermCounter` objects.

This data structure is moderately complicated.  To review, an `Index` object contains a map from each search term to a set of `TermCounter` objects, and each `TermCounter` is a map from search terms to counts.  The method `printIndex` shows how to unpack this data structure:

```java
	public void printIndex() {
		// loop through the search terms
		for (String term: keySet()) {
			System.out.println(term);

			// for each term, print the pages where it appears and the count
			Set<TermCounter> tcs = get(term);
			for (TermCounter tc: tcs) {
				Integer count = tc.get(term);
				System.out.println("    " + tc.getLabel() + " " + count);
			}
		}
	}
```

The outer loop iterates the search terms.  The inner loop iterates the `TermCounter` objects that represent the pages where the search term appears.

*  Run `ant build` to make sure your source code is compiled, and then run `ant Index`.  It downloads two Wikipedia pages, indexes them, and prints the results; but when you run it you won't see any output because we've left one of the methods empty.

Your job is to fill in `indexPage`, which takes a URL (as a String) and an `Elements` object, and updates the index.  The comments below sketch what it should do:

	public void indexPage(String url, Elements paragraphs) {
		// make a TermCounter and count the terms in the paragraphs

		// for each term in the TermCounter, add the TermCounter to the index
	}

*  When it's working, run `ant Index` again, and you should see output like this:

        ...
        configurations
            https://en.wikipedia.org/wiki/Programming_language 1
            https://en.wikipedia.org/wiki/Java_(programming_language) 1
        claimed
            https://en.wikipedia.org/wiki/Java_(programming_language) 1
        servletresponse
            https://en.wikipedia.org/wiki/Java_(programming_language) 2
        occur
            https://en.wikipedia.org/wiki/Java_(programming_language) 2

The order of the search terms might be different when you run it.

Also, run `ant test2` to confirm that this part of the lab is complete.


This lab involved more reading that some of the others, and not as much programming.  But we hope you learned a lot.  Congratulations on getting to the end!
