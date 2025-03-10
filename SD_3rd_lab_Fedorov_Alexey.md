# SD Lab 3: Fuzzing for Software Security Testing

## Completed by Fedorov Alexey (tg: @ullibniss)

---

```
Objective:
This lab aims to introduce students to fuzzing, a dynamic software testing
technique used to discover vulnerabilities in software applications. By the end of
this lab, students will be able to:
1. Understand the basics of fuzzing and its importance in security testing.
2. Set up a fuzzing environment.
3. Use a fuzzing tool to test a sample application.
4. Analyze the results of a fuzzing test.

Prerequisites:
- Basic understanding of programming (Python/C/C++).
- Familiarity with command-line interfaces.
- Basic knowledge of software vulnerabilities (e.g., buffer overflows, crashes).
```

# Lab Setup

## 1. Install AFL

I installed AFL.


## 2. Create a Target Application: Save the following C code as vulnerable.c

Created target application file.

![image](https://github.com/user-attachments/assets/7cee4367-9126-4524-b616-47387b24911e)

## 3. Compile the Target Application with AFL

I compiled application with `afl-gcc`.

![image](https://github.com/user-attachments/assets/47c6c774-c1c4-4598-b8ac-732f6f7a1de6)

# Lab Steps

## Step 1: Understand the Target Application

### Review the vulnerable.c code and identify the potential vulnerability (buffer overflow).

The potential vulnerability is in `vulnerable_function`.

```
void vulnerable_function(char *input) {
    char buffer[100];
    strcpy(buffer, input);
}
```

We can see that the function takes a `char*` pointer as argument and then tries to copy it in buffer of size 100. The main problem here, that we dont know size of input file and if file's size is bigger than buffer's, it will cause crash. 

### Discuss why this vulnerability is dangerous and how it can be exploited.

This vulnerability can be exploited with file that has more than 100 symbols. 

![image](https://github.com/user-attachments/assets/cb0e1617-7ed4-4bdf-b689-a82c87d76195)

It can be used by intruder to crush application.

## Step 2: Run the Fuzzer

### 2.1 Create an input directory for AFL.

I created input directory for AFL.

![image](https://github.com/user-attachments/assets/03ed4153-cb79-4d0d-b6bf-d3d7101de72d)

### 2.{2,3} Run AFL on the target application. Monitor the Fuzzing Process

Let's do it. I started fuzzer. Here is AFL interface.

![image](https://github.com/user-attachments/assets/4547a81f-31de-4e09-a805-fbd7f275792e)

It worked for 9 minutes. Has 235k runs. Found 1 unique crash and no timeouts.

### 2.4 Analyze the Results

I checked `output` directory and found one crash file

![image](https://github.com/user-attachments/assets/fdff4ba4-eb16-4685-832a-e8adb599cf3c)

Let's take a look.

![image](https://github.com/user-attachments/assets/addc3aef-7b98-4b89-a66e-3ceba367059c)

Next sted is replay crash.

![image](https://github.com/user-attachments/assets/9f3103f1-da47-42cb-8216-0f040f2570a1)

Application crashed. This happned because file contains more that 100 symbols.

### 2.5 Fix the Vulnerability

I fixed vulnerability.
```
void safe_function(const char *input) {
    char buffer[100];

    strncpy(buffer, input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';

}
```

I used hardenings from previous Lab to fix this vulnerability. Function `strncpy` takes third argument - size of copying data. This preventes buffer overflow. I also added '\0' to ensure string will properly terminate.

Let's recompile.

![image](https://github.com/user-attachments/assets/3de3dcdb-19cb-44a2-b8b5-080125ef6781)

Now let's test with fuzzer again.

![image](https://github.com/user-attachments/assets/680785f5-04ee-45fb-8d41-7fafdc6afa72)

As we can see, there not crashes after fix.

# Questions

## 1  What is the purpose of fuzzing in software security testing?

The purpose of fuzzing in software security testing is to find vulnerabilities and unexpected behaviors in software by providing it with a large volume of random, malformed, or unexpected inputs. This technique helps to fix issues that might not be detected through traditional testing methods. Fuzzing is particularly effective in finding edge cases and uncovering hidden bugs that could be exploited by attackers.

## 2 How does AFL generate test cases to find vulnerabilities?

AFL generates test cases by using a genetic algorithm to mutate input data and monitor how the program behaves with these inputs. It starts with a set of initial seed inputs and iteratively mutates them, favoring inputs that trigger new code paths or unique behaviors in the program. By instrumenting the target program, AFL tracks code coverage and prioritizes inputs that explore untested or complex areas, increasing the likelihood of discovering vulnerabilities.

## 3 What other types of vulnerabilities can fuzzing detect besides buffer overflows?

Besides buffer overflows, fuzzing can detect a wide range of vulnerabilities, including 
- integer overflows,
- use-after-free errors
- race conditions
- format string vulnerabilities
- memory corruption issues.

It can also find logic errors and denial-of-service (DoS) conditions.

## 4 How can you improve the efficiency of a fuzzing campaign?

I can use high-quality seed inputs, optimizing the fuzzing tool's configuration, and parallel fuzzing to distribute the workload across multiple cores or machines.

## References

- https://en.wikipedia.org/wiki/American_Fuzzy_Lop_(software)
- https://afl-1.readthedocs.io/en/latest/about_afl.html#how-afl-works
