---
title: "Stop Guessing: How to Catch the RIGHT Python Exception"
author: Grace Li
pubDatetime: 2026-01-18T16:00:00Z
postSlug: python-exception-handling-guide
featured: true
draft: false
tags:
  - Python
  - Coding Best Practices
  - Tutorial
description: Don't just "try-except" everything. Learn the detective skills to predict exactly which errors your code might throw.
---

## The "Pokémon" Bad Habit

When new programmers learn `try-except`, they often do this:

```python
try:
    some_risky_code()
except:
    print("Oops, something went wrong.")
```

This is called **"Pokémon Exception Handling"** (*Gotta catch 'em all!*), and it is a **bad practice**. 

Why? Because it catches *everything*—even simple typos or things you *want* to crash your program (like you pressing Ctrl+C). It hides bugs and makes debugging a nightmare.

## The Problem: "How do I know what to catch?"

The reason beginners use the "catch-all" is usually simple: **They don't know what specific error to expect.**

If you write `api_key = st.secrets["GOOGLE_API_KEY"]`, how are you supposed to know that it raises a `KeyError` and not a `ValueError` or `ZeroDivisionError`?

Here are **3 Methods** to predict the future.

---

### Method 1: The "Let It Crash" Method (My Favorite)

This is the easiest way. Don't write the `try-except` block yet. Just run the code and **let it fail**.

If correct `secrets.toml` file is missing, Python will scream at you in the console:

```text
FileNotFoundError: No such file or directory: '.streamlit/secrets.toml'
```

**Aha!** Now you know the name: `FileNotFoundError`.

If the file exists but the key is missing?

```text
KeyError: 'GOOGLE_API_KEY'
```

**Aha!** Now you know the second name: `KeyError`.

Now you can write your code with confidence:

```python
try:
    api_key = st.secrets["GOOGLE_API_KEY"]
except FileNotFoundError:
    # Handle missing file
except KeyError:
    # Handle missing key
```

---

### Method 2: The "Code Detective" Method

Even without running code, the **syntax** often gives you hints.

*   Are you using square brackets `['key']`? → Expect **`KeyError`** (dictionaries) or **`IndexError`** (lists).
*   Are you doing math `/`? → Expect **`ZeroDivisionError`**.
*   Are you opening a file `open()`? → Expect **`FileNotFoundError`** or **`PermissionError`**.
*   Are you converting types `int("text")`? → Expect **`ValueError`**.

---

### Method 3: The "RTFM" Method

**R**ead **T**he **F**riendly **M**anual.

Documentation usually tells you what errors a function raises. For example, the Python docs for requests say: *"In the event of a network problem (e.g. DNS failure, refused connection, etc), Requests will raise a `ConnectionError` exception."*

## 📝 Cheat Sheet: Common Exceptions

Reference this table for the most common errors you'll encounter.

| Action | Likely Exception | Why? |
| :--- | :--- | :--- |
| `my_dict["missing_key"]` | `KeyError` | The key doesn't exist in the dictionary. |
| `my_list[100]` | `IndexError` | You tried to grab an item at an index that doesn't exist. |
| `int("hello")` | `ValueError` | The function received the right type (string) but wrong value. |
| `print(unknown_var)` | `NameError` | You haven't defined this variable yet. |
| `None.some_method()` | `AttributeError` | You're trying to use a method on `None` (very common!). |
| `10 / 0` | `ZeroDivisionError` | You can't divide by zero. |
| `open("ghost.txt")` | `FileNotFoundError` | The file isn't where you think it is. |
| `import fake_module` | `ModuleNotFoundError`| You forgot to `pip install` or misspelled the name. |

## Summary

1.  **Never** blindly use bare `except:`.
2.  **Let it crash** locally to see the real error name.
3.  Catch **only** what you can handle, and let the rest crash (so you can fix the real bugs!).
