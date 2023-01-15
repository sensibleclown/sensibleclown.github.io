---
title: Two numbers addition
date: 2023-01-15 08:00:00 +/-TTTT
categories: [rust]
tags: [rust, programming, algorithm]     # TAG names should always be lowercase
---

## Problem
A function is required that accepts a non-empty array of unique integers and a target sum integer as input. If any two integers in the input array add up to the target sum, the function should return them in any order within an array. If no two integers in the array sum up to the target sum, the function should return an empty array. It's important to note that the target sum can only be obtained by adding two distinct integers from the array and it's not possible to use a single integer twice to obtain the target sum.

## Alternative 1
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

## Alternative 2
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

## Alternative 2
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


