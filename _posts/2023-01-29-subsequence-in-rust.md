---
title: Subsequence in Rust
date: 2023-01-29 08:00:00 +/-TTTT
categories: [programming, algorithms]
tags: [rust, programming-challenge, easy] # TAG names should always be lowercase
---

A subsequence in an array is a group of numbers that retain their sequence order, but may not be consecutively placed. For example, [1, 3, 4] is a subsequence of [1, 2, 3, 4] and [2, 4] is also a valid subsequence. It's worth noting that both a single number and the entire array can also be considered as subsequences.

## Solution

This code is a Rust implementation of a function called "is_subsequence" that takes in two arguments, a mutable reference to an array of integers. It returns a boolean.

### Solution 1

```rust
// The `is_subsequence` function checks if `sequence` is a subsequence of `array`.
// It takes two arguments: `array` and `sequence`, both slices of integers `&[i32]`.
// The function returns a boolean indicating if `sequence` is a subsequence of `array`.

fn is_subsequence(array: &[i32], sequence: &[i32]) -> bool {
    // Initialize a variable `j` to keep track of the position in `sequence`
    let mut j = 0;

    // Iterate over elements in `array`
    for &i in array {

        // Check if `j` is within bounds of `sequence` and if `i` is equal to
        // `j`-th element in `sequence`
        if j < sequence.len() && i == sequence[j] {

            // Increment `j` if both conditions are true
            j += 1;
        }
    }

    // Return true if all elements in `sequence` were found in `array`,
    // false otherwise
    j == sequence.len()
}

```

## Test Case

```rust
#[cfg(test)]
mod tests {
    use super::is_subsequence;

    #[test]
    fn test_is_subsequence() {
        let array: [i32; 8] = [5, 1, 22, 25, 6, -1, 8, 10];
        let sequence: [i32; 4] = [1, 6, -1, 10];
        let result = is_subsequence(&array, &sequence);
        assert_eq!(result, true);

        let array: [i32; 8] = [5, 1, 22, 25, 6, -1, 8, 10];
        let sequence: [i32; 1] = [26];
        let result = is_subsequence(&array, &sequence);
        assert_eq!(result, false);
    }
}
```
