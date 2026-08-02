Which exception will be thrown?


try {
throw new RuntimeException("fail"); } finally {
}
throw new Exception("from finally"); }


💡 The Big Picture & Analogy

The answer is Exception("from finally"). The RuntimeException("fail") thrown in the try block disappears completely, with no trace left behind.

Think of it like this: you're about to mail an important letter (the RuntimeException)
but right before it leaves your hand, someone rushes in and says "wait, I have something more urgent," then throws a different letter (the Exception from finally) instead. 
The first letter never gets delivered, and nobody ever reads it.



🧩 Why Do We Need It? (Why this behavior happens)

finally is designed to always run, regardless of whether try completes normally or throws an exception
it's the place where you do cleanup, like closing connections or files.

The problem is: what if the finally block also throws an exception? 
The JVM has to decide which exception to propagate to the caller, because a method can only throw one exception at a time
and the JVM always chooses the latest one, which is the one from finally.

⚙️ Core Logic & How It Works

Here's the JVM's step-by-step flow:

Enter try → hits throw new RuntimeException("fail") → prepares to propagate this exception
Before the exception can leave the method, the JVM must run the finally block first
Inside finally, it hits throw new Exception("from finally") → a new exception is created
When a new exception is thrown inside finally, it immediately overrides/discards the exception that was pending
The method ultimately throws only Exception("from finally") — the original RuntimeException("fail") is gone, with no stack trace left to show it ever happened

⚖️ Trade-offs & When to Use
This isn't a pattern you should ever use intentionally — throwing an exception inside finally is widely considered an anti-pattern.

Why it's dangerous: it makes debugging extremely difficult, because the real exception that explains the root cause (fail) silently vanishes, 
leaving only a fake exception (from finally) that tells you nothing about the actual problem.
When you're likely to hit this unintentionally: when writing cleanup code inside finally, like connection.close() or stream.close(), 
and those methods happen to throw a checked exception themselves — very common in older codebases that don't use try-with-resources.

🛠️ Real-World Scenario / Mini Example
A common real-world case:

[example java]
try {
    doSomeBusinessLogic(); // throws NullPointerException("user id is null")
} finally {
    connection.close(); // throws SQLException("connection already closed")
}

The logs the team sees will show SQLException: connection already closed
everyone chases a bug in the connection layer, when the real problem was user id is null all along. 
Time wasted debugging the wrong thing entirely.

How to prevent it: use try-with-resources (for anything implementing AutoCloseable). 
It handles this automatically through a mechanism called suppressed exceptions
the second exception gets attached to the first one (accessible via getSuppressed()) instead of silently replacing it.

[example java]
try (Connection conn = getConnection()) {
    doSomeBusinessLogic();
} // conn is closed automatically; if close() throws, it becomes a suppressed exception

🧠 Lead's Key Takeaway
Golden Rule: Never let a finally block have the chance to throw an exception on its own. 
If code inside finally might throw (like closing a resource), wrap it in an inner try-catch — or better yet, 
always use try-with-resources when working with anything that implements AutoCloseable.

===========================

## 🧩 Why Do Many People Say "Both Can Happen"?

This confusion usually comes from 3 main sources:

### 1. Mixing Up Suppressed Exceptions with Plain try-finally

Many people know Java has a **suppressed exceptions** mechanism (where both exceptions get preserved together), but this mechanism only works with **try-with-resources** — not with a plain `try-finally` block.

[example java]
// This case = suppressed exception IS preserved
try (AutoCloseable r = ...) {
    throw new RuntimeException("fail");
} // if r.close() also throws, it gets attached as "suppressed"

// The original scenario = NO suppression at all, the first one is just gone
try {
    throw new RuntimeException("fail");
} finally {
    throw new Exception("from finally");
}


Someone who learned about suppressed exceptions might mistakenly assume this mechanism also applies to plain `try-finally`. 
In reality, **it only kicks in with try-with-resources** — in a regular `try-finally`, there's no suppression mechanism at all. The first exception vanishes with zero trace.

### 2. Confusing "Actually Occurred" with "Propagated to the Caller"
Some people interpret it as "both exceptions are thrown/created" — which is technically true in the sense that the JVM does construct both exception objects. 
But the actual question is which exception **will be thrown** (i.e., propagated out to the caller), and the answer to that is only one: `Exception("from finally")`.

In short:
- The first exception **is genuinely created** ✅
- But it **never propagates out to the caller** ❌ — it gets discarded before it ever gets that far

### 3. Confusion from Other Languages' Behavior
Some languages (like certain versions of Python, or .NET with `InnerException`) automatically chain exceptions together, 
which leads developers coming from those languages to assume Java does the same. But Java's plain `try-finally` doesn't do this — you'd need try-with-resources to get that chaining behavior.

## 🧠 Lead's Key Takeaway
**Golden Rule:** Always separate two questions that sound similar but aren't the same:
- Which exception **gets created**? → multiple exceptions can be created
- Which exception **gets propagated to the caller**? → only ever one. In a plain `try-finally`, the one from `finally` always wins — no exceptions to this rule.
