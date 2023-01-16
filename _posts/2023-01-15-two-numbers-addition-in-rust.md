---
title: Two numbers addition in Rust
date: 2023-01-15 08:00:00 +/-TTTT
categories: [rust]
tags: [rust, programming, algorithm]     # TAG names should always be lowercase
---

## Problem
A function is required that accepts a non-empty array of unique integers and a target sum integer as input. If any two integers in the input array add up to the target sum, the function should return them in any order within an array. If no two integers in the array sum up to the target sum, the function should return an empty array. It's important to note that the target sum can only be obtained by adding two distinct integers from the array and it's not possible to use a single integer twice to obtain the target sum.

## Solution
This code is a Rust implementation of a function called "two_number_sum" that takes in two arguments, a mutable reference to a vector of integers array and an integer target_sum. It returns a vector of integers.

### Solution 1
```rust
use std::collections::HashSet;

fn two_number_sum(array: &mut Vec<i32>, target_sum: i32) -> Vec<i32> {
    let mut s = HashSet::new();

    for i in array {
        let match1: i32 = target_sum - *i;
        if s.contains(&match1) {
            return vec![*i, match1];
        }

        s.insert(i);
    }

    vec![]
}
```

The function uses a HashSet, a data structure from the standard library's std::collections module, to keep track of integers in the input array.

The code starts by initializing an empty HashSet called s. Then, it iterates over each element in the input array using a for loop.

For each iteration, the code calculates match1 which is the target sum minus the current element in the array.

Then, it checks if the calculated match1 is already present in the HashSet using the contains method, if match1 is present it means that we have found a pair of elements in the array that add up to the target sum, so the function returns a vector containing these two elements.

If match1 is not present in the HashSet, the current element is inserted into the HashSet using the insert method.

After the for loop completes, if no pair of elements that add up to the target sum is found, the function returns an empty vector.

The use of a HashSet allows the function to efficiently check if an element is present in the array and is a more efficient way than using nested loops as it allows for constant time lookups.

### Solution 2
```rust
fn two_number_sum(array: &mut Vec<i32>, target_sum: i32) -> Vec<i32> {
    array.sort();
    let mut left = 0;
    let mut right = array.len() - 1;

    while left < right {
        let current_sum = array[left] + array[right];
        if current_sum == target_sum {
            return vec![array[left], array[right]];
        } else if current_sum < target_sum {
            left += 1;
        } else if current_sum > target_sum {
            right -= 1;
        }
    }
    vec![]
}
```
The code starts by sorting the input array in ascending order using the sort() method. Then it initializes two pointers, left and right . left is set to the start of the array and right to the end of the array.
The function then enters a while loop that continues until the left pointer is less than the right pointer. Inside the while loop, the function calculates the current sum by adding the elements at the left and right pointers. 

If the current sum is equal to the target sum, the function returns a vector containing the elements at the left and right pointers. 
If the current sum is less than the target sum, the left pointer is incremented.
If the current sum is greater than the target sum, the right pointer is decremented.

After the while loop completes, if no pair of elements that add up to the target sum is found, the function returns an empty vector.

This is a implementation of Two pointer algorithm to find two number sum in a array which matches target sum, which is a O(n) time complexity algorithm.

### Solution 3
```rust
fn two_number_sum(array: &mut Vec<i32>, target_sum: i32) -> Vec<i32> {
    for i in 0..array.len() - 1 {
        for j in i + 1..array.len() {
            if array[i] + array[j] == target_sum {
                return vec![array[i], array[j]];
            }
        }
    }
    vec![]
}
```
The function starts by using two nested for loops to iterate over all pairs of elements in the input array. The outer loop iterates over the elements starting at index 0, and the inner loop iterates over the elements starting at an index one greater than the current index of the outer loop.

For each pair of elements, the function checks if their sum is equal to the target sum. If it is, the function returns a vector containing these two elements.

If the loops complete without finding a pair of elements that add up to the target sum, the function returns an empty vector.

This code will check all possible pairs of elements in the array. As the number of pairs is n(n-1)/2 where n is the number of elements in the array, the time complexity of this algorithm is O(n^2) and it may not be the most efficient solution for large arrays.

## Test
```rust
#[cfg(test)]
mod tests {
    use super::two_number_sum;

    #[test]
    fn test_two_number_sum() {
        let mut array = vec![3, 5, -4, 8, 11, 1, -1, 6];
        let target_sum = 10;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![-1, 11]);

        let mut array = vec![3, 5, -4, 8, 11, 1, -1, 6];
        let target_sum = 10;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![-1, 11]);

        let mut array = vec![1, 2, 3, 4, 5, 6, 7, 8];
        let target_sum = 9;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![1, 8]);

        let mut array = vec![4, 4];
        let target_sum = 8;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![4, 4]);

        let mut array = vec![4, 4, 4];
        let target_sum = 8;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![4, 4]);

        let mut array = vec![1, 2, 3, 4, 5, 6, 7, 8];
        let target_sum = 20;
        let mut result = two_number_sum(&mut array, target_sum);
        result.sort();
        assert_eq!(result, vec![]);
    }
}
```


