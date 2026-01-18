---
title: "Why Use Logging When Print Works?"
author: Grace Li
pubDatetime: 2026-01-18T15:00:00Z
postSlug: python-logging-vs-print-guide
featured: false
draft: false
tags:
  - Python
  - Coding Best Practices
  - Tutorial
description: Why print() is bad for production, how logging saves the day, and a fun breakdown of log levels for beginners.
---

## The "Print Statement" Trap

We’ve all been there. You have a bug. You don't know why. So you do this:

```python
print("Checking variable x...")
print(x)
print("Did we get here? Yes.")
```

It works! But then you finish your code, deploy it, and suddenly your console is **flooded** with "Checking variable x..." messages that you forgot to delete. Or worse, you have an error in production, but you can't see what happened because you deleted all those print statements to clean up!

**There is a better way.** Enter: **The `logging` module.** 🦸‍♀️

---

## Why Logging is Better than Print

Think of **`print()`** like shouting across the room. everyone hears it, all the time.
Think of **`logging`** like a professional radio channel. You can tune in to different frequencies (channels) depending on what you need to know.

| Feature | Print() | Logging |
| :--- | :--- | :--- |
| **Control** | All or Nothing | Tunable (Show only errors? Show everything?) |
| **Context** | Just text | Adds timestamps, filenames, and line numbers automatically |
| **Destination** | Console only | Console, Files, Email, Cloud... anywhere! |

---

## The "Radio Channels" (Log Levels)

Python's logging has 5 main "frequencies" or levels. Imagine you are teaching a classroom:

1.  **DEBUG (`logging.debug`)**: *The Whisper.*
    > "I am currently holding the chalk." "I am moving my hand left."
    > *Use this for detailed diagnostics that you only care about when fixing a bug.*

2.  **INFO (`logging.info`)**: *The Announcement.*
    > "Class has started." "We are on page 34."
    > *Use this for general events to prove things are working normally.*

3.  **WARNING (`logging.warning`)**: *The Stern Look.*
    > "Jimmy, stop talking."
    > *Something unexpected happened, but the class (program) keeps going.*

4.  **ERROR (`logging.error`)**: *The Shout.*
    > "The fire alarm is ringing!"
    > *A specific function failed/crashed.*

5.  **CRITICAL (`logging.critical`)**: *The Explosion.*
    > "The school is on fire!"
    > *The entire application is crashing.*

---

## How to Use It (The Code)

Here is the magic snippet to get you started. Put this at the top of your file:

```python
import logging

# 1. Configure the "Global Switch"
# level=logging.INFO means "Ignore debug whispers, show me everything else"
logging.basicConfig(level=logging.INFO)

# 2. Create your "Personal Channel"
# __name__ automatically uses your filename (e.g., 'src.ai_agent')
logger = logging.getLogger(__name__)
```

Now, instead of print, you do this:

```python
def calculate_grade(score):
    logger.info(f"Calculating grade for score: {score}")

    if score < 0:
        logger.error("Score cannot be negative!")
        return None
    
    # This won't show up unless you change config to DEBUG!
    logger.debug("Parsing complex math equation...") 
    
    print("This is a bad habit!") # Don't do this!
```

---

## Pro Tip: Changing the Channel

The beauty is that you can change the behavior **without changing your code**.

If you run your app normally:
```python
logging.basicConfig(level=logging.WARNING)
```
*Result:* Silence. Peace and quiet. Only real problems show up.

If you are debugging a crash:
```python
logging.basicConfig(level=logging.DEBUG)
```
*Result:* You see EVERYTHING. Every tiny calculation. 

You didn't have to add or delete a single print statement. You just turned the volume knob. 🎛️

## Pro Tip: Dynamic Logging (The Magic Trick) 🎩

"But wait!" you say. "I still have to change the code to switch between `WARNING` and `DEBUG`!"

Not if you use **Environment Variables**. This is how the pros do it.

```python
import os
import logging

# 1. Get the level from the environment (default to INFO if not set)
env_level = os.getenv("LOG_LEVEL", "INFO").upper()

# 2. Configure logging with that level
logging.basicConfig(level=env_level)
```

Now, you can change the behavior just by how you **run** the program:

*   **Normal Run:** `python my_app.py` → (Defaults to INFO)
*   **Debug Run:** `LOG_LEVEL=DEBUG python my_app.py` → (Shows all debug logs!)

No code changes required. Just pure magic. ✨

---

## Summary

*   **Print** is for YOU (temporary debugging).
*   **Logging** is for the APP (long-term health).
*   Use `INFO` for "it's working", and `ERROR` for "it broke".

Happy Coding! 🚀
