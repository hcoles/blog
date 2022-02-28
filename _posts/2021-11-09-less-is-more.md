---
layout: post
title:  "Less is more"
author: henry
categories: [pitest, java]
image: assets/images/less_is_more.png
description: "Less is more"
featured: true
hidden: false
---

## Mutation testing, the fun bit

Mutation operators are kind of fun. When someone sits down to write a new mutation tool, the bit they'll be excited about is how they're going to create the mutants. When someone first starts using pitest, despite all the advice to the contrary, they'll be tempted to enable all the operators.

But more isn't always **better**. Like most things in software engineering, picking a set of operators is a balancing act where we have to make trade-offs. 

The trade-offs seem obvious: the more mutants we create, the more confidence we have in our test suite. But on the flip side, the more mutants we have, the longer our analysis time will be, and the more likely we are to encounter equivalent or junk mutants (especially if experimental or research operators are enabled).

So there are good, obvious, reasons why we might not want to enable all the operator. But actually, it is more complex than that. The first part of the trade off is deceptive. Having more mutants does not (necessarily) mean we have done a better job of checking our test suite. 

More mutants can be, quite literally, a complete waste of time due to something called **mutant subsumption**.

### When more is good

Before we talk about subsumption, lets look at when having more mutants *is* good.

Here's a bit of toy java code.

```java
  public List<Path> doStuff(Collection<Path> paths) {
    return paths.stream()
      .filter(Files::exists)
      .filter(this::checkPath)
      .collect(Collectors.toList());
  }

  boolean checkPath(Path path) {
    // bit of random logic
    return Files.size(path) != 0;
  }
```

If we were to run pitest against this with just the false returns mutator enabled, it would create the following mutants (shown here both seeded into the code at once for brevity).

```java
  public List<Path> doStuff(Collection<Path> paths) {
    return paths.stream()
      .filter(f -> false) // (FR1) 
      .filter(this::checkPath)
      .collect(Collectors.toList());
  }

  boolean checkPath(Path path) {
    // bit of random logic
    return false; // (FR2)
  }
```

Clearly, these two mutants do not give us much confidence. A test suite that passed in only empty collections of paths (or collections of non existent files) would not detect these mutants, but any test that supplied a collection with a file the existed and passed the logic in `checkPath` would kill them both.

We would gain more confidence if we also enabled the `TRUE_RETURNS` mutator. This adds the following mutants :-

```java
  public List<Path> doStuff(Collection<Path> paths) {
    return paths.stream()
      .filter(f -> true) // (TR1) 
      .filter(this::checkPath)
      .collect(Collectors.toList());
  }

  boolean checkPath(Path path) {
    // bit of random logic
    return true; // (TR2)
  }
```

Pitest has effectively removed the first filter by mutating the lambda the compiler generates for the `Files::exists` method reference, and has effectively removed the second filter by forcing `checkPath` to always return true. Killing these mutants requires us to also pass in paths that do not exists and fail the logic in `checkPath`.    

Lets add some more operators and see if our confidence increases further. The `EMPTY_RETURNS` operator adds the following mutant :-

```java
  public List<Path> doStuff(Collection<Path> paths) {
    paths.stream()
      .filter(f -> true)
      .filter(this::checkPath)
      .collect(Collectors.toList());
    return Collections.emptyList(); // (ER1)
  }
```

But this gives no additional value. If the other mutants didn't exist it would be useful. It would ensure we had a test that passes in a non empty collection of Paths, and checked the output. But when the other mutants are present it provides no benefit. You could not write a test that passed when `ER1` was present, but did not also pass when `FR1`, or `FR2` were present.

And this is the problem with creating lots of mutants. Having more mutants is guaranteed to create more mundane practical issues, such as slower analysis and information overload, but it doesn't guarantee more confidence in your test suite. And if you are tracking a codebase's mutation score (which I do not recommend) it can be misleading. You can make the score go **up** by enabling more operators.

## Subsumption

In the academic literature, the phenomenon where a mutant is made pointless by the presence of other mutants in called 'subsumption'. It's a tricky thing to get your head around, because the rules for which mutant subsumes another are not always straightforward or global.

For the following code, using the same three operators, the `EMPTY_RETURNS` mutants would *not* be subsumed.

```java
public List<String> logic(int i) {
  if (someLogic(i) {
    return List.of("Foo", "Bar"); // could mutate here
  }
  return List.of("Cats"); // could mutate here
}

boolean someLogic(int i) {
  return i != 42 && i != 7; // could mutate to true or false
}
```

The examples so far are particularly difficult to reason about as they involve mutants in different parts of the code. Things are a bit easier when we look at mutants that affect the same instruction. For these mutants more consistent rules sometimes exist.

For the simple boolean expression

```java
  boolean mutateMe(int i) {
    return i < 42;
  }
```

The `stronger` mutator set in pitest 1.7.3 will generate the following mutants

* return i <= 42
* return true
* return false

Other mutations are possible, such as

* return i > 42
* return i >= 42

But pitest doesn't include these as there would be no point. The three existing mutations are more "stable", you could never write a test suite where tests failed for those mutants but passed for `i > 42` or `i >= 42`.

So the primary way in which pitest avoids creating useless subsumed mutants, is by using stable sets of mutators by default. This is a very blunt approach though, as we can see if we look more closely at the three mutants we do create.

To fully exercise the conditional statement we need to supply three classes of values

* i less than 42
* i greater than 42
* i exactly equal to 42

The truth table for our original code, and our three mutated versions looks like this.


<table class="subsumption">
<thead>
<tr>
<th></th>
<th>&lt; 42</th>
<th>&lt;= 42</th>
<th>true</th>
<th>false</th>
</tr>
</thead>
<tbody>
<tr>
<td>41</td>
<td>T</td>
<td><strong>T</strong></td>
<td><strong>T</strong></td>
<td>F</td>
</tr>
<tr>
<td>42</td>
<td>F</td>
<td>T</td>
<td>T</td>
<td><strong>F</strong></td>
</tr>
<tr>
<td>43</td>
<td>F</td>
<td><strong>F</strong></td>
<td>T</td>
<td><strong>F</strong></td>
</tr>
</tbody>
</table>

If you peer at this very closely and think for a while, you can see that the `always true` mutant is subsumed by the combination of the other two, and is therefore useless.

`<=` gives the same value as `<` for two of the inputs. If you only feed in 41 and 43, then `<=` and `<` seem to behave the same. If you feed `always false` 41 it gives the wrong answer, but it is the only mutant to give the right answer for 42.

Between them, `<=` and `always false` can give the same answers as the unmutated code for all classes of input, the `always true` mutator just duplicates what `<=` already does.

We could also say that the combination of `always true` and `always false` subsumes `<=`, but `<=` looks more like a bug a programmer might actually make for this code, so it is this mutant we choose to keep.

So perhaps we shouldn't enable the `always true` mutator?

It is, of course, not as simple as that. If our original code had been 

```java
boolean mutateMe(int i) {
  return i <= 42;
}
```

Then, if we refer back to our table, we can see that it would be the `always false` mutation that would be subsumed.

So we need a sharper tool.

# Introducing Subsumption Analysis

I've talked before about the [pitest extensions](https://www.arcmutate.com/) I have been developing with Group CDG. They now include a new plugin that adds new mutation operators, but also adds subsumption analysis to reduce the overall number of mutations.

The result is a new stronger mutation analysis, which seeds faults into code that pitest cannot, while keeping the number of mutants low and the signal to noise ration high.

If you'd like to try it out, details of our beta programme are on the [Arcmutate site](https://www.arcmutate.com/). 
