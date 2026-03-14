---
title: "Mastering Python Unit Testing: Setup and Mocking"
author: Grace Li
pubDatetime: 2026-01-19T23:00:00Z
postSlug: python-unittest-guide
featured: false
draft: false
tags:
  - Python
  - Coding Best Practices
  - Tutorial
description: A comprehensive guide to setting up Python unit tests and using unittest.mock to isolate dependencies.
---

## Introduction

Unit testing is the practice of verification—ensuring that individual parts of your code work as expected in isolation. While many libraries exist (like `pytest`), Python's built-in `unittest` module is the standard foundation that every developer should master.

## Basic Setup

To write a unit test, you create a dedicated file (usually starting with `test_`) and define a class that inherits from `unittest.TestCase`.

### 1. The Structure

```python
import unittest

# The function we want to test
def add(a, b):
    return a + b

class TestMathOperations(unittest.TestCase):
    
    def test_addition(self):
        result = add(2, 3)
        # Assertions verify the outcome
        self.assertEqual(result, 5)

if __name__ == '__main__':
    unittest.main()
```

### 2. Common Assertions

The `TestCase` class provides methods to check specific conditions.

| Assertion | Checks For |
| :--- | :--- |
| `assertEqual(a, b)` | `a == b` |
| `assertTrue(x)` | `bool(x) is True` |
| `assertIn(item, list)` | `item` is in `list` |
| `assertRaises(Error)` | The code block raises a specific Exception |

## Mocking Dependencies

Real-world code often interacts with external systems: databases, APIs, or the file system. Unit tests should be fast and isolated, so we use **Mocks** to fake these interactions.

The `unittest.mock` library is the standard tool for this.

### 1. Using `patch` to Fake Functions

Suppose we have a function that calls an expensive API:

```python
# app.py
def get_user_data(api_client, user_id):
    # Imagine this makes a slow network call
    return api_client.fetch(user_id)
```

We can mock `api_client` so we don't actually hit the network.

```python
from unittest.mock import MagicMock
import unittest

class TestUserData(unittest.TestCase):
    
    def test_get_user_data(self):
        # Create a fake object
        mock_api = MagicMock()
        
        # Configure what it returns when called
        mock_api.fetch.return_value = {"name": "Grace", "id": 1}
        
        # Call function with the fake
        result = get_user_data(mock_api, 1)
        
        # Verify result is what the mock returned
        self.assertEqual(result['name'], "Grace")
        
        # Verify the function actually CALLED the mock correctly
        mock_api.fetch.assert_called_once_with(1)
```

### 2. Patching Imports

Sometimes you need to replace a library directly. Use the `@patch` decorator.

```python
from unittest.mock import patch
import my_module

# If my_module.py uses requests.get()
@patch('my_module.requests.get')
def test_api_call(self, mock_get):
    # Configure the fake response
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"key": "value"}
    
    # Run code
    result = my_module.fetch_data()
    
    # Verify
    self.assertTrue(result)
```

## Running Tests

You can run all tests in your directory from the command line:

```bash
python -m unittest discover tests
```

This command automatically finds any file matching the pattern `test_*.py` inside the `tests/` folder and runs it.
