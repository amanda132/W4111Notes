# I. General Idea: What and Why
Quick review on steps for a new application

* Requirements Analysis: What are you going to build? 
* Conceptual Database Design: Pen-and-pencil description, ER diagram
* Logical Database Design: Conversion into a relational database schema
* Schema Refinement: Problem fixing, **Normalization** (to reduce redundancy by decompositions to normal form)
* Physical Database Design: Optimize for speed/storage


# II. Redundancy and Anomalies
In general, redundancy is bad. We want to find a systematic way to solve this problem.
#### Example 2.1: 
* people have names and addresses 
* hobbies have costs
* many-to-many relationship between people and hobbies

|   sid(key)    | name           |  address  |  hobby  |   cost   |
| ------------- |:--------------:|:---------:|:-------:|---------:|
|       1       |       Eugene   | amsterdam |  trucks |  $$      |
|       1       |       Eugene   | amsterdam |  cheese |  $       |
|       2       |       Bob      |   40th    |  paint  |  $$$     |
|       2       |       Bob      |   40th    |  cheese |  $       |
|       4       |      Shaq      |  florida  |swimming |  $       |

Some information are stored repeated, e.g. (Eugene, amsterdam), (Bob, 40th), (cheese, $).
These redundancies lead to anomalies (inconsistence):

* Update Anomaly
  * In Example: If we update Eugene's address, we need to change every record related to Eugene. The same for Bob.
  * Definition: If one copy of such repeated data is updated, an inconsistency is created unless all copies are similarly updated.
* Insertion Anomaly
  * In Example: Suppose the primary key is (sid, hobby), it's not allowed to add a person without hobby.
  * Definition: It may not be possible to store certain information, unless some other information is stored as well.
* Deletion Anomaly: 
  * In Example: If we delete a hobby e.g. swimming, Shaq's record will be deleted unless it's allowed to update all 'swimming' to 'NULL'.
  * Definition: It may not be possible to delete certain information without losing some other information.

* A Possible Approach:
we can decompose a table into smaller table. Unfortunately, not all decompositions are good. For example, if we decompose the above table into person(sid, name, addr, cost) and personhobby(sid, hobbyname), then it is not clear which cost goes to which hobby, as the two tables only have one common field -- sid.

# III. Decomposition
1. What is decomposition?
2. What criteria of decomposition do we care?
3. Potential Problems with Decompositions

### 1. What is decomposition?
Fortunately, many problems arising from redundancy can be addressed by replacing a relation **R** with a collection of 'smaller' relations,
s.t.
* each contain subset of attributes in **R**,
* and together include all attributes in **R**.
For example, a relation with attributes (ABCD) can be replaced with (AB,BCD) or (AB, BC, CD).

### 2. What criteria of decomposition do we care?
#### Example 3.1: A bad attempt

|   sid(key)    | name           |  address  |   cost   |
| ------------- |:--------------:|:---------:|---------:|
|       1       |       Eugene   | amsterdam |  $$      |
|       1       |       Eugene   | amsterdam |  $       |
|       2       |       Bob      |   40th    |  $$$     |
|       2       |       Bob      |   40th    |  $       |
|       4       |      Shaq      |  florida  |  $       |

|   sid(key)    |  hobby  |
| ------------- |:-------:|
|       1       |  trucks |
|       1       |  cheese |
|       2       |  paint  |
|       2       |  cheese |
|       4       |swimming |

This is a bad attempt because you loose information in the decomposition, making it a lossy decomposition. Additionally it uses the same amount of memory to store the data so there is no benefit to decomposing the table in this way. 

Explanation: If we read 'person' relation only, we cannot understand the meaning of attribute 'cost' without join, since it's the hobby's cost. However, if we equi-join two relations on sid, it actually cannot recover the original relation. 
Because we have two tuples with 'sid = 1', equi-join is ambiguous that for instance cost '$$' can be associated with 'cheese' or 'trucks'. The decomposition breaks the original association between hobby and cost, and thus loses information. We call such decomposition as a 'lossy decomposition'.

#### Example 3.2: A familiar attempt

![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-13%20at%203.40.00%20PM.png?raw=true)

It works out, thanks to heuristic ER diagram.

Explanation: In ER model, attributes only describe the entity they belongs to, and every entity table has a primary key that determines all values of a tuple. In this sense, both person and hobby are independent entities with meaningful attributes. The relationship table describes the association between person and hobby, and if we join it with person and hobby, we will recover the original table without any loss of information.

#### Desirable criteria of decomposition
* Lossless-join: enable us to recover original relation from the smaller relations. **π<sub>X</sub>(R)⋈π<sub>Y</sub>(R) = R**
* Dependency-preservation: enable us to enforce any constraint on the original relation by simply enforcing some constraints on each of the smaller relations, i.e. to enforce constraints even without joins.

#### Decompose or not? A trade-off
* Advantages
  * Less anomalies: We make changes in one place that apply everywhere, and it eliminates possibility of data getting “out of sync”.
  * Direct and fast query: By accessing a sub-section of the data for some queries
* Disadvantages
  * If always need all data together, joins may be slower
  * Less flexible because there is no redundancy 

#### How can we systematically decompose relations according to the desirable criteria?
See below: functional dependencies and normal forms

### 3. Potential Problems with Decompositions
* **Lossy-join**: able to recover R from smaller relations
* **Non-dependency-preserving**: constraints on R hold by only enforcing constraints on smaller schemas
* **Performance**: additional joins, may affect performance

# IV. Functional Dependency
1. What is FD?
2. How do we use FD?
3. Where to find FD?

### 1. What is FD?
**Definition** Let X, Y be sets of attributes. If for any tuples t1 and t2 that t1[X] = t2[X] implies t1[Y] = t2[Y], we then call X->Y a functional dependency.

Intuitively, FD X->Y means that if two tuples agree on the values in attributes X, they mush also agree on the values in attributes Y. 

#### Example 4.1: FDs


|   sid(key)    | name           |  address  |  hobby  |   cost   |
| ------------- |:--------------:|:---------:|:-------:|---------:|
|       1       |       Eugene   | amsterdam |  trucks |  $$      |
|       1       |       Eugene   | amsterdam |  cheese |  $       |
|       2       |       Bob      |   40th    |  paint  |  $$$     |
|       2       |       Bob      |   40th    |  cheese |  $       |
|       4       |      Shaq      |  florida  |swimming |  $       |

* sid -> name, address 
* hobby -> cost
* sid, hobby -> name, address, cost

For `sid -> name, address`, sid sufficient to identify name and address, but not hobby.
In other words, if 2 records have the same sid, their name and address are the same.

#### Example 4.2: 
| a | b | c | d |
| - |:-:|:-:| -:|
| 1 | 1 | 1 | 1 |
| 2 | 2 | 2 | 1 |
| 2 | 3 | 4 | 1 |
| 3 | 4 | 5 | 1 |
| 4 | 5 | 6 | 1 |

This table is not consistent with the following FDs:

a -> bc, c -> d.

#### Example 4.3: primary key and redundancy
| a | b | c | d | e |
| - |:-:|:-:|:-:| -:|
| 1 | 2 | 2 | cat | 1 |
| 1 | 2 | 2 | dog | 2 |
| 2 | 3 | 3 | dog | 2 |

Suppose the FDs are:

a -> bc, d->e.

Here, neither ``a'' nor ``d'' can be a primary key. Then we can add data to make the above table clearly redundant. For example, if we add one row 

3 | 4 | 4 | dog | 2,

then dog and 2 shows redundancy.
#### Note that
* A functional dependency is a integrity constraint. It is a statement about _all_ instances of relations.
* Redundancy depends on FDs (SUPER IMPORTAANT): 
  * FD X->Y definition does not require that X be minimal. X can be either super key or candidate key.
  * If K is candidate key of R, then K -> R. 
  * When K is not a table key, then it can be copied around, so it can introduce redundancy.
  * In other words, if R has no redundancy, FDs are key constraints.

### 2. How do we use FDs (at a high level)
* Start with some relational schema
* Find out its FDs
* Use these to design a better schema, with less redundancy and anomalies.

### 3. Where to find FDs?
We can never deduce that an FD does hold by looking at one or more instances of the relation, because an FD, like other ICs, is a statement about all possible legal instances of the relation.

### 4. How to Use FD's to find Keys (Tabular Method)

There is a tabular method to finding keys (even composite ones) using these Function Dependencies (FD's). Take a look at the following example:

#### Example 4.4

R(ABC)

F = {A --> B, B --> C}

R is a relation containing the attributes A,B,C and F is the set of FD's that corresponds to this relation. In order to find all of the keys in relation R, you could set up a table like this:

| **L**        | **M**           | **R**  |
| ------------- |:-------------:| -----:|
| A     | B | C |

L means **left**, M means **middle**, and R means **right**. Look through each FD and check if each attribute only appeared on the left hand side of a FD, only on the right-hand side, or on both sides. In the above example, A only appeared the left hand side of a FD, so it goes into the "L" column. Similarly, C went into the "R" column. Since B appeared on the right-hand side of one FD and on the left-hand side of another, it belongs to the middle column. 

So what does this table means? Well, here is the interpretation:

**Left:** all attributes in this column must be part of every key

**Right:** attributes in this column do not belong to any key

**Middle:** these attributes may or may not be part of a key (more example below to find out what they really are)

For the above example, if we perform A+ (A closure), we would get ABC, giving us all of the attributes of relation R. Therefore, A is certainly a key for this relation.

Keys: A

Attributes used for keys: A

Attributes not used for keys: B, C

#### Example 4.5

R(ABCD)

F = {AB --> C, C --> B, C -->D}

Let's set up our handy table:

| **L**        | **M**           | **R**  |
| ------------- |:-------------:| -----:|
| A     | B C | D |

Using our table, we see that A might be a key. To be sure, we will perform A+, which will yield...just A by reflexivity. Therefore, A on its own is not a key, and must add another attribute to it. **This is where the middle column comes into play.** Let's try another closure like AB+:

AB --> AB (reflexivity)

AB --> C (given)

C --> D (given)

Therefore, AB+: ABCD

Aha! Since we have all of the attributes of R, we now know that AB is a key. Now let's use the other middle attribute C:

AC --> AC (reflex)

C --> B (given)

C --> D (given)

Therefore, AC+: ACBD

Now we find that AC is also a key. Therefore, we conclude for this problem:

Keys: AB, AC,

Attributes used for keys: A, B, C

Attributes not used for keys: D

#### Example 4.6

R(ABC)

F = {A --> B, B --> C, C --> A}

| **L**        | **M**           | **R**  |
| ------------- |:-------------:| -----:|
|      | A B C |  |

This situation is rather painful, but it is doable. Since all of the attributes are in the middle column, we must do multiple permutations of closure to find the keys (ie: closures of 1 attribute at a time,  2 attributes at a time, and 3 attributes at a time).

A+: ABC (A is a key)

B+: ABC (B is a key)

C+: ABC (C is a key)

In this case, all of the attributes by themselves are all keys, so we are done.

Keys: A, B, C

Attributes used for keys: A, B, C

Attributes not used for keys: null

# V. Normal Forms
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-13%20at%2011.36.51%20PM.png?raw=true)

If a relation is in a certain normal form (BCNF, 3NF etc.), it is known that certain kinds of problems are avoided/minimized. This can be used to help us decide whether decomposing the relation will help.

Hence we will focus on two normal forms: BCNF and 3NF
* BCNF has no redundancy, but may lose some dependencies (Everything depends on “the key, the whole key, and nothing but the key”)
* 3NF preserves all dependencies but has some redundancy (A lossless-join, dependency-preserving decomposition of R
￼into a collection of 3NF relations is always possible.)

## 1. BCNF: Boyce-Codd Normal Form
* BCNF ensures that no redundancy occurs from Functional Dependencies (other forms redundancy can occur)
* In ensuring no redundancy, BCNF may not preserve all dependencies (ie. constraints may need to be enforced with joins)

### (1). Definition:
Given a relation *R* and a set of functional dependencies *F* of the form (*X* -> *A*), we say that *R* is in BCNF if:

```
for every X -> A in F:
  (A is in X OR 
   X is a superkey of R)
```

In words:
* If **every** functional dependency is either trivially satisfied (ie. A in X, right-side of arrow is also part of the left-side) or a superkey of all corresponding relations, then the decomposition is in BCNF
 * That is, loop through FDs and if any do not satisfy the "OR" condition above, then decomposition is not in BCNF
  * This is how we "check" if a relation is in BCNF
* Conversely, if **any** functional dependency is neither trivially satisfied nor a superkey, then the entire decomposition is **NOT** in BCNF
* **NOTE:** Superkey: a combination of attributes that can uniquely identify an entity (row) in a relation (table).
 * So a trivial superkey would just be a tuple of every attribute and candidate keys are a subset of superkeys, since they are *minimal* superkeys
 * More importantly, we say an attribute is a superkey of a relation w.r.t. FDs iff every attribute in the relation is defined in the functional dependency (eg. given `ABC` and `A -> BC`, `A` is a superkey of `ABC` because `A` uniquely identifies `B` and `C` as stated in the FD `A -> BC`)
 * If you can not check a FD in a relation, it defaults to be true

### (2). A Few Examples:
#### Example 5.1
** Given the relation `ABCD` and the FDs `A -> B`, `A -> C`**

| Relations | BCNF? | Reason |
| ------ | ------ | ------ |
| ABCD | NO | `A -> B` is not trivially satisfied (ie. `B` in not in `A`) and `A` is not a superkey of `ABCD` |
| AB, AC, AD | YES | `A` is superkey of `AB` and `AC` (again don't really care about `AD`) |
| AB, AC, D | YES* | **BUT** this is **NOT** desirable because we've lost data about how `D` relates to the rest `A`, `B`, and `C` |
| ABC, D | YES* | `A -> B` and `A -> C` implies `A -> BC`, which is satisfied by this relation; but like the relation above, this is **NOT** desirable because we've lost data about `D` |

So why is `AB, AC, AD` considered a "good" decomposition while `AB, AC, D` is "bad"?
* Because we know `A` is a key of `AB` and `AC`, we can accurately recover the original schema `ABCD` by performing a join on `A`
* Conversely, we cannot do this on `AB, AC, D` because after joining `AB` and `AC` on `A` we have no lossless way of combining `ABC` and `D` (ie. we can only perform a cross-product, which will most likely produce too many tuples)

#### Example 5.2
**Given the relation `SHNAC` and the function dependencies `S -> NA` and `H -> C`**

| Relations | BCNF? | Reason |
| ------ | ------ | ------ |
| SHNAC | NO | for `S -> NA`, `NA` is not in `S` and `S` is not a superkey of `SHNAC` => NOT BCNF |
| SNA, SHC | NO | for `S -> NA`, `S` is a superkey of `SNA`... ignore `SHC` because does not contain `NA`... for `H -> C`, ignore `SNA`... `H` not superkey of `SHC` => NOT BCNF (ie. `H -> C` FD is not satisfied) |
| SNA, HC, SH | YES | for `S -> NA`, `S` is a superkey of `SNA`... ignore `HC` and `SH`... for `H -> C`, ignore `SNA`... `H` is a superkey of `HC`... ignore `SH` => YES BCNF (ie. all FDs satisfied!) |

#### Example 5.3
**Given the relation `IJKLM` and the functional dependency `IJ -> K`**

| Relations | BCNF? | Reason |
| ------ | ------ | ------ |
| IJKLM | NO | Should be pretty obvious by now: `IJ` is not a superkey of `IJKLM` (and `K` is not in `IJ`) |
| IJK, KLM | YES* | `IJ` is a superkey of `IJK`, but this is **NOT** desirable because we cannot recover the original table |
| IJK, IJLM | YES | `IJ` is a superkey of `IJK` and we can perform a lossless join on `IJ` |

**Ok what if we add the functional dependency `L -> M`?**

| Relations | BCNF? | Reason |
| ------ | ------ | ------ |
| IJK, IJLM | NO | Now, because of our new FD `L` must also be a superkey which isn't true for `IJLM` |
| IJK, LM | YES* | Again, technically this is in BCNF, but it is **NOT** desirable because we lose data about how `IJK` relates to `LM` and therefore can't accurately recover the original table |
| IJK, LM, IJL | YES | `IJ` is superkey of `IJK`, `L` is superkey of `LM`, and we can perform a lossless join using `IJL` |

Things to note about this last example:
* We can kind of see a pattern begin to emerge for BCNF:
 * A simple decomposition could just be a relation for each FD and then an additional relation of all the keys if there is no common key between the two tables
 * So for this last example: `IJK` and `LM` are relations for each FD and `IJL` is just a table of the keys
* We can also begin to see a rough outline of the algorithm for decomposing a relation to BCNF
 * Basically, if the relation isn't in BCNF, just keep breaking the relation up along the lines mentioned above until the relation is in BCNF

### (3). An informal BCNF Decomposition Algorithm
#### Algorithm
```
while BCNF is violated: 
   R with FDs FR
   if X->Y violates BCNF
      turn R into R-Y & XY
```
Intuitively, we can understand this as, "while BCNF is violated, find a relation that is violating a FD and decompose it further by splitting it into two tables: one with all the attributes defined in the FD and another with everything else and the key of the FD (ie. left side of FD = `X`).

#### Example 5.4: Step By Step Example
**Given the relation `BCNO` and the FDs `BC -> N` and `N -> BO`, decompose R using the following steps:**
 1. `BCNO` violates BCNF
 2. Break `BCNO` (in this example we use `N -> BO` as the "base" FD)
  * `NBO` is one relation (obtained by just combining attributes of "base" FD)
  * The other relation is simply `R - Y` = `BCNO` - `BO` = `CN`
 3. Now we have `NBO`, `CN` which is in BCNF

**NOTE:** that we have "lost" `BC -> N` since there is no longer a relation that contains all three attributes, but this is ok because we are in BCNF and apparently no one cares :/

### (4). A formal BCNF Decomposition Algorithm
#### Algorithm
```
    DecomposedRelations = { R }

    while DecomposedRelations is not empty
      R = pop a relation from DecomposedRelations
      FR = subset of FDs in R's projection
      for fd X->Y in FR
        if fd violates BCNF with respect to R
          NewRs = decompose R using fd 
          add NewRs to DecomposedRelations
          continue while loop
```
#### Example 5.5: Step By Step Example
```
    R:   ABCDE
    FDS: A->BCDE, BC->A, D->B, C->D

    # suppose we first check D->B:

    D doesn't determine R, thus we decompose into: BD, ACDE

    Let's check BD:
    
      Only D->B is in its projection, thus its in BCNF.  
      
    Let's check ACDE:
    
      Only C->D is in its projection, so ACDE is redundant.
      Decompose into: CD, ACE

    ACE has empty projection, thus it's in BCNF
```
## 2. 3rd Normal Form (3NF)
We relax BCNF (BCNF is a stricter version of 3NF):

    F: a set of functional dependencies over relation R
        for (X->Y) in R:
            Y is in X OR 
            X is a super key of R OR 
            Y is part of a key in R


This new condition is NOT trivial, as this key is minimal, not a superkey. This guarantees lossless join AND dependency preserving decomposition

Tradeoff is that a relation in 3NF form does retain some redundancies. 
#### Example 5.6
Schema: SBDC
SBD -> C,S -> C is not in 3NF;
SBD -> C,S -> C, C->S is.
In both cases, SC is stored redundantly.

#### Example 5.7: a pizza example
![](https://github.com/agango/Scribesnotes-image/blob/master/image.png)
  
The key for this table is (Pizza, Type)-unique pairing of attributes, each pizza has one of every type
We have the functional dependencies Topping->Type, and Pizza, Type->Topping. This relation is not in BCNF form, but is in 3NF form because the FD Topping->Type is no longer violated, as Type is a part of the key. However, this table does have redundancies.

I thought we were trying to get rid of redundancies?

Yes, but we can improve our data design abilities by understanding redundancy 

# VI. Formalization of Normal Forms
## 1. Closure of FDs
Let's start with an example: 
If we know that Name->BDay, and BDay->Age, then we know that Name->Age

A functional dependency f' is implied by a set F if f' is true when F is true. 

All the functional dependencies that can be implied from F is called the closure (F<sup>+</sup>)

We can construct a closure of a set F using Armstrong's axioms

## 2. Armstrong's Axioms:
* **Reflexivity**: if attribute Y is a subset of key X, then X->Y
    The "trivial" rule-a column always determines itself, and a set of column always determines a subset of that set
    * A->A
    * (X, Y)->Y
* **Augmentation**: if X->Y, then XZ->YZ for any Z
    * A->B, C->C by reflexivity. So, (A,C)->(B,C)
* **Transitivity**: if X->Y and Y->Z, then X->Z
    * Apply them in sequence. If we know (x, y), we know that from X=x we can get Y=y. Similarly, if we know (y, z), we know that from Y=y we can get Z=z. Therefore, from X=x we can get Y=y, and since we know y, we can get Z=z. 

These rules are sound and complete.
* **Sound**: doesn’t produce FDs not in the closure.
* **Complete**: doesn’t miss any FDs in the closure.

#### Example 6.1
Consider the functional dependency F=(A->B, B->C, CB->E)
Is A->E in the closure?
Solution: 
* A->B (given) 
* A->AB (augmenting A on both sides, AA can be reduced to A-remember that a functional dependency is a constraint between two sets of attributes in a relation, and a set requires distinct elements)
* A->BB (apply A->B)
* A->BC (apply B->C)
* BC->E (given)
* A->E (transitivity)

We can generate some rules from Armstrong's axioms. 

**Union**: If A->B and A->C, then A->BC (from A->B, we can augment C to both sides to get AC->BC, and substitute A for C to get A->BC)

**Decomposition**: If A->BC, then A->B and A->C (this is also standard form). 

**Pseudo-transitivity**: If A->B, and CB->D, then CA->D holds (We can prove this using augmentation. We can augment C to both sides of A->B to get CA->CB, and use transitivity to get CA->D). 

#### Example 6.2

Consider the following relation scheme R=(A, B, C, G, H, I), and the set of functional dependencies A->B, A->C, CG->H, CG->I, B->H.

We want to make sure that our closure includes all non-trivial functional dependencies implied by the dependencies given above.

Our closure, F+, is then the following:

-A->H, by transitivity

-CG->HI. To see why, consider augmenting I to both sides of CG->H. We then get CGI->HI. However, we have CG->I, so we can substitute CG for I, and because functional dependencies deal with sets of attributes, CGCG->HI becomes CG->HI.

-AG->I. We can augment G to both sides of AG->CG, because CG->I, we get AG->I. It is important to note that we could do this in one step using pseudo-transitivity (A->C, so we can substitute A for C in CG->I to get AG->I).

## 3. Minimum Cover of Functional Dependencies

Closures can allow us to compare sets of FD's meaningfully. 

For example, if F1={A->B, A->C, A->BC}, and F2={A->B, A->C), are they equivalent? Yes! We can get from F2 to F1 using Armstrong's axioms, so F1 is not minimal.

If there's a closure (a maximally expanded set of functional dependencies), then there must be a minimal set as well. 

### Steps:
 1. Turn the FD's into _standard form_
    * Split the FD's so there is only one attribute on the right side
 2. Minimize left side of each FD
    * For each FD, check if we can delete a left attribute using another FD. 
        * Given (A, B, C)->D, and B->C, we can reduce this to (A, B)->D, and B->C
        * Why? B->C, so we can replace C on the left with B, and since we're looking at a set of attributes, this collapses to (A, B)->D
 3. Delete redundant functional dependencies
    * Check each remaining functional dependency and check to see if it can be deleted (if it's in the closure of the other FD's)

**NOTE: You must do the steps in this order**

Minimum covers are not necessarily unique

#### Examples 6.3
Consider the set A->B, ABC->E, EF->G, ACF->EG

1. First convert to standard form (only one attribute on the right side):
    * We get A->B, ABC->E, EF->G, **ACF->E**, **ACF->G**
2. Minimize left side
    * A->B, **AC->E**, EF->G, ACF->E, ACF->G
        * Why? ABC->E and A->B implies AC->E
3. Delete redundant FD's
    * A->B, AC->E, EF->G
        * ACF->E implied by AC->E (a smaller set of attributes that determines E), ACF->G implied by AC->E EF->G (we can substitute AC for E to get ACF->G to reconstruct this dependency)


#### Examples 6.4
Consider the set ABC -> DE, A -> DB, B -> DC.

1. First convert to standard form:
    * **ABC ->D**, **ABC ->E**, **A ->D**, **A ->B**, **B->D**, **B->C**.
2. Minimise left side
    * **A ->D**, **A ->E**, A ->D, A ->B, B->D, B->C.
3. Remove Redundant ones:
    * A ->E, A ->B, B->D, B->C.

## 4. Principled Decomposition

We eventually want to be able to decompose relation R into sub-relations R1, R2,..., Rn such that we can do a join on R1, R2, ..., Rn and get back the original relation R, while preserving the functional dependencies F of R. 

However, we've seen issues with decomposition:
   
    * Lost joins (can't recover R from R1, R2, ... , Rn)
    
    * Lost dependencies (we cannot enforces all the dependencies when we decompose R into the sub-relations)

### (1). How can we tell if a decomposition will yield lossless joins?

Consider the decomposition of R into R1 and R2

A decomposition is lossless w.r.t F if the closure of F contains one of the following:

(R1 intersect R2) -> R1 OR (R1 intersect R2) -> R2. 

This means that if the intersection of R1 and R2 is a key for either R1 or R2, and is contained in the closure of F, then the decomposition of R into R1 and R2 is lossless. 

**Note: here R1 intersect R2 is simply the intersection of the attributes in the relation. If R1=(A, B) and R2=(B, C), then R1 intersect R2 = B**

#### Example 6.5

![](https://github.com/agango/Scribesnotes-image/blob/master/example2.png)

Why is this an example of lossy decomposition? In this case, (R1 intersect R2) = AB intersect BC = B. In table R1, B does not determine anything (we have 2 values of A for the same value of B), and the same for table R2. As we can see, when we join the two relations, we get new rows that were not there before. We have not been able to completely recover the original relation by joining R1 and R2. 

What's correct?

![](https://github.com/agango/Scribesnotes-image/blob/master/example3.png)

Why is this correct?

R1 intersect R2 = A, so we get the FD A -> AC (which can be decomposed to A->A, which is trivial, and A->C, which is a FD of R). This functional dependency is in the closure of F. 

**Note:** Just as a quick note, the difference between a BCNF decomposition that loses a functional dependency, and an invalid BCNF decomposition is the lossless join component. Even if we have lost an FD in the new relations, if they can be joined together to recover the original relation, then they are in BCNF. However, a bad or invalid BCNF decomposition will suffer from lossy decomposition. 
 
### (2). What about dependency preservation?

Let F<sub>R</sub> be the projection of F onto R

Then F<sub>R</sub> is the set of FD's X->Y in F closure such that X and Y are attributes of the new relation R'. 

If we decompose the original relation R into X and Y, then the decomposition is dependency preserving if all closures of all the FD's in all the new relations (in this case, X and Y) is equal to closure of F over R (a FD X->Y is preserved in a relation R if R contains all the attributes of X and Y).

#### Example 6.6

Consider the following:

R(A, B, C, D) under F = {A → B, B → C}. 

A valid BCNF decomposition would be R1(AB), R2(AC), R3(AD).

However, this decomposition is not dependency preserving, because while A->B is preserved in R1, B->C is not preserved anywhere. 

We can instead decompose this into R1(AB), R2(BC), R3(AD)

Here, both FD's are preserved (A->B preserved in R1, B->C preserved in R2), so this decomposition is dependency preserving.

#### Example 6.7

Consider ABCD, C is the key, AB->C, D->A

BCNF decomposition: BCD, DA

AB->C doesn't apply to either of the new relations, so this decomposition is not dependency preserving. 

## (3). Decomposing into 3NF

We have already seen above the algorithm for BCNF decomposition. Just to refresh:

```
while BCNF is violated: 
   R with FDs FR
   if X->Y violates BCNF
      turn R into R-Y & XY
```

For a 3NF decomposition: 

```
First find the minimal cover Fmin
Run BCNF on Fmin
For X->Y in Fmin not in projection onto R1,.., Rn
    create relation X->Y
```

**Notes:** Based on the solution of homework & final, the algorithm for a 3NF decomposition is:

```
1. Got one of the BCNF decomposition (Let's call it BCNF1)
2. First find the minimal cover Fmin
3. Check each FD in Fmin on BCNF1:
   For X->Y in Fmin not in projection onto R1,.., Rn
       create relation X->Y
4. Remove redundant tables. (Redundant tables are those whose attributes are a subset of another table in the decomposition)
```

#### Example 6.8

**Given the relation `BCNO` and the FDs `BC -> N` and `N -> BO`, decompose R using the following steps:**
 1. `BCNO` violates BCNF
 2. Break `BCNO` (in this example we use `N -> BO` as the "base" FD)
  * `NBO` is one relation (obtained by just combining attributes of "base" FD)
  * The other relation is simply `R - Y` = `BCNO` - `BO` = `CN`
 3. Now we have `NBO`, `CN` which is in BCNF
 4. Fmin = `BC->N, N->B, B->O`
 5. Check `NBO`, `CN` using Fmin, we found the FD `BC->N` is lost
 6. Add `BC->N` back to the schemas, `NBO`, `CN` and `BCN`
 7. Since `CN` is a subset of `BCN`, remove redundant table `CN`
 8. Final results: `NBO`, `BCN`

#### Example 6.9

Relation `ABCDE`, FDs `A->BCDE`, `B->C`, `D->EC`.

```
1. Applying `B->C`, `BC`, `ABDE`
2. Applying `D->E` (projection of `D->EC` on `ABDE`), `BC`, `DE`, `ABD`
3. Fmin = { A->B, A->D, B->C, D->E, D->C }
4. `D->C` is missing
5. 3NF: `BC`, `DE`, `ABD`, `DC`
```

#### Example 6.10
Relation `ABCDE`, key `A`, `BC ->A`, `D->B`, `C->D`
1. Get Fmin:
* A->B, A->C, A ->D, A ->E, BC ->A, D->B, C->D
* A->B, A->C, A ->D, A ->E, C ->A, D->B, C->D (since C->D and D->B)
* A->B, A->C, A ->D, A ->E, C ->A, D->B (since C->A and A->D)
2. BCNF:
* DB,ACDE using D->B
* DB,AC,ADE using A->C
* DB, CD, AE, AD using A->E
3. Recover dependency:
* DB,AC,AE,AD,AB to recover lost dependency A->B.

## (4). Practice problem and strategy to find minimal cover
Find minimal cover for R(ABCDE) F={A->D, BC->AD, C->B, E->A, E->D}

*1. singleton right hand-side
```
break BC->AD to BC->A and BC->D
now, F={A->D, BC->A, BC->D, C->B, E->A, E->D}
```

*2. check if there exist extraneous attributes on right hand-side
```
strategy: do it one FD at a time.
	(1) BC->A
		To do this, we check what is B+ and C+. If any of B+ or C+ include the A, we can eliminate the other one.
		B+: B(by reflexivity)
		C+: C(by reflexivity), B(by C->B), A(by BC->A), D(by BC->D). Because C+ includes B, we can eliminate B from left hand-side
		now, F={A->D, C->A, BC->D, C->B, E->A, E->D}
	(2) BC->D
		As above, we can eliminate B
		now, F={A->D, C->A, C->D, C->B, E->A, E->D}
```
*3. eliminate any redundant dependency
```
strategy: do it one FD at a time, pretend if it does not exist and see if we can get the FD by other FDs.
	(1) A->D
		A+ without this FD: A(by reflexivity). No way to get A->D w/o this FD. KEEP it.
	(2) C->A
		C+ without this FD: C(by reflexivity), D(by C->D), B(by C->B). No way to get C->A w/o this FD. KEEP it.
	(3) C->D
		C+ without this FD: CADB. We can get C->D w/o this FD. ELIMINATE it.
	(4) C->B
		C+ without t1. * his FD: CAD. KEEP it.
	(5) E->A
		E+ without this FD: ED. KEEP it.
	(6) E->D
		E+ without this FD: EAD. ELIMINATE it.
```


## Code to solve FD problems 

**NOTE THIS IS STARTER CODE AND INCORRECT!!!**: 
https://www.instabase.com/hooshmand68/comsw4111Normalization/fs/Instabase%20Drive/main.ipynb
