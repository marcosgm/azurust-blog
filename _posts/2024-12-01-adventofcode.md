---
layout: post
title:  "Solving Advent of Code using Rust"
date:   2024-12-01 23:19:54 -0500
categories: rust advent of code
---
Today I want to challenge me a bit more with Rust so I'll try the [Advent of code 1st challenge](https://adventofcode.com/2024/day/1)

It's a good exercise to learn about vectors and iterators, and returning values as tuples

## 1st part of the puzzle

Given two lists: left and right:
| Left | Right |
|------|------|
|  3   |  4   |
|  4   |  3   |
|  2   |  5   |
|  1   |  3   |
|  3   |  9   |
|  3   |  3   |
Pair up the numbers and measure how far apart they are. Pair up the smallest number in the left list with the smallest number in the right list, then the second-smallest left number with the second-smallest right number, and so on.

Within each pair, figure out how far apart the two numbers are; you'll need to add up all of those distances. For example, if you pair up a 3 from the left list with a 7 from the right list, the distance apart is 4; if you pair up a 9 with a 3, the distance apart is 6.

What is the total distance between your lists?

## The solution to the 1st puzzle

``` rust
use std::fs::File;
use std::io::{self, BufRead};

fn main() -> io::Result<()> {
    let path = "input.txt";
    let (left, right) = load_numbers_from_file(&path)?;

    // Print the loaded numbers
    for (num1, num2) in left.iter().zip(right.iter()) {
        println!("{} {}", num1, num2);
    }

    // Create new copies of the left and right vectors, but sorted in ascending order
    let mut left_sorted = left.clone();
    let mut right_sorted = right.clone();
    left_sorted.sort();
    right_sorted.sort();

    // Now calculate the distance for every pair of numbers in left_sorted and right_sorted, incrementing the total_distance count on every iteration
    let mut total_distance = 0;
    for (num1, num2) in left_sorted.iter().zip(right_sorted.iter()) {
        total_distance += (num1 - num2).abs();  // Calculate the absolute difference between the two numbers and add it to the total_distance
    }
    // Explanation from Copilot
    // the Zip method in  `left.iter().zip(right.iter())` is used to combine two iterators into a single iterator of pairs. 
    // - `left.iter()`: This creates an iterator over the elements of the `left` vector.
    // - `right.iter()`: This creates an iterator over the elements of the `right` vector.
    // - `zip`: This method takes two iterators and returns a new iterator that yields pairs of elements, with one element from each of the original iterators.
    // In the context of the code, `left.iter().zip(right.iter())` creates an iterator that yields pairs of elements from the `left` and `right` vectors.
    // For example, if `left` contains `[1, 2, 3]` and `right` contains `[4, 5, 6]`, the resulting iterator will yield `(1, 4)`, `(2, 5)`, and `(3, 6)`.
    // This is useful for iterating over two collections in parallel.

    println! ("Total distance: {}", total_distance);

    Ok(())
}

fn load_numbers_from_file(path: &str) -> io::Result<(Vec<i32>, Vec<i32>)> {
    // Open the file
    let file = File::open(&path)?;

    // Create a buffered reader
    let reader = io::BufReader::new(file);

    // Vectors to store the numbers
    let mut left: Vec<i32> = Vec::new();
    let mut right: Vec<i32> = Vec::new();

    // Read the file line by line
    for line in reader.lines() {
        let line = line?;
        // Split the line by whitespace and parse the numbers
        let mut parts = line.split_whitespace();
        if let (Some(first), Some(second)) = (parts.next(), parts.next()) {
            if let (Ok(num1), Ok(num2)) = (first.parse::<i32>(), second.parse::<i32>()) {
                left.push(num1);
                right.push(num2);
            }
        }
    }

    Ok((left, right))
}
```

## 2nd part of the puzzle

Calculate a total similarity score by adding up each number in the left list after multiplying it by the number of times that number appears in the right list.
The first number in the left list is 3. It appears in the right list three times, so the similarity score increases by 3 * 3 = 9. The third number in the left list is 2. It does not appear in the right list, so the similarity score does not increase (2 * 0 = 0).
What is the total similarity score for all the items in the list?

## The solution to the 2nd puzzle

I add some more code to the main() function

``` rust
    // Calculate the similarity score using the original left and right vectors
    let (leftsim, rightsim) = create_similarity_score(&left, &right)?;
    let mut total_score = 0;
    for num in leftsim.iter() {
        total_score += num;
    }
    // no need to add up the score from the rightsim vector
    println! ("Similarity score: {}", total_score);
    Ok(())


    
fn create_similarity_score(left: &Vec<i32>, right: &Vec<i32>) -> io::Result<(Vec<i32>, Vec<i32>)> {
    let mut leftsim = Vec::new();
    let mut rightsim = Vec::new();

    for &num in left.iter() {
        leftsim.push(addup_ocurrences(&num, right));
    }

    for &num in right.iter() {
        rightsim.push(addup_ocurrences(&num, left));
    }

    Ok((leftsim, rightsim))
}

fn addup_ocurrences(entry: &i32, list: &Vec<i32>) -> i32 {
    // Add up all the times a number shows up in a list
    let mut addup = 0;
    for num in list.iter() {
        if num == entry {
            addup += entry;
        }
    }
    addup
}
```