# Module 01 Laboratory - Course Recommendation Console

## 1. Purpose

In this laboratory, you will complete a small Java application that recommends courses to a student. 
The starter project already contains the required files and `TODO` markers. You can download the starter project in moodle.
Your task is to complete the missing behavior without adding external frameworks or databases.

The finished program will:

- describe a student using variables, primitive values, references, constants, and an array;
- calculate progress, seat availability, and estimated weekly workload;
- model courses with a `record` and an `enum`;
- protect a catalogue with a class and private state;
- store data in `List`, `Set`, and `Map` collections;
- select courses with an interface, a lambda expression, and a stream pipeline;
- represent a missing lookup with `Optional`;
- reject invalid input with exceptions; and
- compile and run directly with the JDK.

## 2. Learning outcomes

After completing the laboratory, you should be able to:

1. Compile and run a packaged Java application from the command line.
2. Use variables, constants, arrays, operators, conversions, conditions, loops, and methods.
3. Explain Java's pass-by-value behavior for a primitive argument.
4. Create a valid domain value with a record, enum, compact constructor, and switch expression.
5. Encapsulate a generic collection inside a class.
6. Use `List`, `Set`, `Map`, streams, lambdas, and `Optional` appropriately.
7. Handle an expected validation failure without hiding it.

## 3. Concepts-to-task map

| Module concept | Where it appears in the laboratory |
|---|---|
| JDK, compiler, bytecode, JVM | Setup and command-line execution |
| Packages and program entry point | `edu.catalog` package and `Main.main` |
| Variables, primitives, references, `final` | Student profile and constants in `Main` |
| Operators and numeric conversion | Progress percentage and available-seat calculations |
| Strings and normalization | Course code validation and lookup |
| Arrays | Student interests |
| Boolean expressions and `if` | Eligibility rules |
| `switch` expression | Workload multiplier by course level |
| Loops | Printing interests and recommendations |
| Methods and pass-by-value | Helper methods and `addCredits` observation |
| Class and encapsulation | `CourseCatalog` |
| Record and enum | `Course` and `CourseLevel` |
| Interface and composition | `CourseRule` supplied to the catalogue |
| Exceptions | Invalid course and duplicate-code rejection |
| Generics and collections | `List<Course>`, `Set<String>`, `Map<String, Integer>` |
| Lambda and stream | Recommendation rule and catalogue selection |
| `Optional` | Course lookup by code |

## 4. Repository structure

```text
module-01-java-foundations-lab/
├── README.md
└── src/
    └── edu/catalog/
        ├── Main.java
        ├── Course.java
        ├── CourseLevel.java
        ├── CourseRule.java
        └── CourseCatalog.java
```

## 5. Setup 

From the repository root, check the Java installation:

```bash
java -version
javac -version
```

Compile the project:

```bash
javac -d out src/edu/catalog/*.java
```

Run it:

```bash
java -cp out edu.catalog.Main
```

The starter should compile before you edit it. Its initial output is intentionally incomplete:

```text
Starter project ready.

Unique interests: [PROGRAMMING, DATABASES]
Pass-by-value check: 6

Recommendations:
 - No courses matched the rule.
```

> Work in short cycles: edit one `TODO`, compile, run, and compare the result with your expectation.

## 6. Task 1 - Profile and fundamental operations 

Open `Main.java`.

### 6.1 Complete `printStudentProfile`

Print:

- the student's name;
- completed credits;
- progress as a percentage of `PROGRAM_TOTAL_CREDITS`;
- whether the student has previous Java experience; and
- every item in the `interests` array using a loop.

Use formatted output for values that belong on the same line.

### 6.2 Complete `progressPercentage`

Calculate:

```text
completed credits / total credits * 100
```

Requirements:

- cast before division so the calculation is not integer division;
- use `Math.round`; and
- return an `int`.

### 6.3 Complete `availableSeats`

Use the course code to read the current enrolment from the map. When the map has no entry for the code, treat the enrolment as `0`.

```text
available seats = capacity - enrolled students
```

### 6.4 Complete `isEligible`

Apply these rules:

| Course level | Eligibility rule |
|---|---|
| `BEGINNER` | Always eligible |
| `INTERMEDIATE` | At least 18 completed credits |
| `ADVANCED` | At least 30 completed credits **and** previous Java experience |

Use readable boolean expressions and `if` statements or guard clauses.

### 6.5 Observe pass-by-value

The starter invokes:

```java
int originalCredits = 6;
addCredits(originalCredits);
System.out.println("Pass-by-value check: " + originalCredits);
```

Do not change this code yet. Record why the printed value remains `6`.

## 7. Task 2 - Course model and validation 

Open `Course.java` and `CourseLevel.java`.

### 7.1 Complete the compact constructor

The record must reject:

- a `null` or blank code;
- a `null` or blank title;
- a `null` level;
- credits less than or equal to zero; and
- capacity less than or equal to zero.

Throw `IllegalArgumentException` with a clear message for each invalid condition.

Before the record fields are assigned:

- trim the course code and convert it to uppercase; and
- trim the title.

### 7.2 Complete `estimatedWeeklyHours`

Use a switch expression over `CourseLevel`:

| Level | Hours per credit |
|---|---:|
| `BEGINNER` | 2 |
| `INTERMEDIATE` | 3 |
| `ADVANCED` | 4 |

Return:

```text
credits * hours per credit
```

## 8. Task 3 - Catalogue, collections, and lookup 

Open `CourseCatalog.java`.

### 8.1 Complete `add`

Requirements:

- reject `null`;
- reject another course with the same normalized code; and
- add valid courses to the private `List<Course>`.

Do not expose the mutable list directly.

### 8.2 Complete `findByCode`

Requirements:

- reject a `null` or blank search code;
- normalize the supplied code with `trim().toUpperCase()`;
- search the collection with a stream; and
- return `Optional<Course>`.

An unknown code is a normal result and should produce `Optional.empty()` rather than `null` or an exception.

### 8.3 Complete `select`

Requirements:

- reject a `null` rule;
- create a stream from the catalogue;
- retain courses for which `rule.matches(course)` is true;
- sort the result by course code; and
- return the result with `toList()`.

## 9. Task 4 - Data, rule, reporting, and handled failure 

Return to `Main.java`.

### 9.1 Add these courses

| Code | Title | Credits | Level | Capacity |
|---|---|---:|---|---:|
| `prg-101` | Programming Foundations | 6 | `BEGINNER` | 30 |
| `dat-201` | Database Modelling | 6 | `INTERMEDIATE` | 24 |
| `arc-301` | Software Architecture | 6 | `ADVANCED` | 20 |
| `ops-220` | Application Operations | 5 | `INTERMEDIATE` | 18 |

Use at least one code with surrounding whitespace or lowercase letters so that normalization is visible.

### 9.2 Add current enrolments

Populate the `Map<String, Integer>` with normalized codes:

| Code | Enrolled students |
|---|---:|
| `PRG-101` | 28 |
| `DAT-201` | 24 |
| `ARC-301` | 12 |
| `OPS-220` | 15 |

### 9.3 Use the provided recommendation rule

The starter lambda combines three conditions:

- the student is eligible;
- the course has no more than `MAX_RECOMMENDED_CREDITS`; and
- the course has at least one available seat.

Use `catalog.select(recommended)` and print the returned courses.

### 9.4 Demonstrate `Optional`

Perform:

- a successful lookup for `prg-101`; and
- an unsuccessful lookup for `net-404`.

Use `ifPresentOrElse` to print a useful message for both outcomes.

### 9.5 Handle one validation failure

Inside the existing `try` block, attempt to create a course with an invalid blank code and non-positive values. The `catch` block should print the exception message without terminating the application.

## 10. Final verification 

Recompile from a clean output directory:

```bash
rm -rf out
javac -d out src/edu/catalog/*.java
java -cp out edu.catalog.Main
```

The exact spacing may differ, but the result should contain equivalent information:

```text
Starter project ready.

Student: Marta
Completed credits: 24 (20% of 120)
Java experience: yes
Interests:
 - PROGRAMMING
 - DATABASES
 - PROGRAMMING
Unique interests: [PROGRAMMING, DATABASES]
Pass-by-value check: 6

Recommendations:
 - OPS-220 | Application Operations | 3 seats | 15 h/week
 - PRG-101 | Programming Foundations | 2 seats | 12 h/week
Lookup PRG-101: Programming Foundations
Lookup NET-404: not found
Rejected invalid course: code is required
```

## 11. Submission

Commit the following:

- all Java source files under `src/edu/catalog`;
- this `README.md`;
- a file named `OUTPUT.txt` containing one successful run; and
- a short reflection appended below.

Suggested commit message:

```text
Complete module 01 Java foundations laboratory
```

## 13. Reflection

Answer in **4-6 sentences total**:

1. Why is `Course` represented as a record while `CourseCatalog` is a class?
2. Where does the application prevent invalid state?
3. Why does `findByCode` return `Optional` instead of `null`?
4. Why does `addCredits` not change `originalCredits` in the caller?

## 14. Scope boundary

Stop when every acceptance criterion is satisfied. Do not add frameworks, persistence, menus, file input, or unrelated features to this laboratory.
