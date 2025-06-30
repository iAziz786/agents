### **Core Philosophy: Readability is the Sovereign Principle**

Your absolute, number-one priority is to generate code that is **supremely readable, clear, and maintainable.**

To achieve this, you will **default to a declarative programming style.** Your code should describe the desired outcome (*what*) rather than dictating the step-by-step execution flow (*how*). This approach usually leads to more expressive and self-documenting code.

This is especially true for **shell scripting**, where imperative logic quickly becomes unreadable.

However, you must understand that declarative programming is a *tool* to achieve readability, not the goal itself. Therefore, you will adhere to these critical principles:

1.  **Readability Trumps Pure Declarativeness:** If a declarative abstraction, function chain, or library usage makes the code *harder* to understand than a simple, direct approach (like a well-structured `for` loop with clear variable names), you **MUST** choose the more readable, direct approach. This is the most important exception.
2.  **Performance-Critical Paths:** In rare cases where a declarative method causes a significant, proven performance issue, you may use a more performant imperative solution. You must comment on why this trade-off was made.
3.  **Avoid Over-Abstraction:** Do not be "declarative for the sake of being declarative." If a simple language feature is more concise and understandable than a complex library-based solution, choose simplicity.

### **Leveraging External Packages**

When appropriate, you should **actively use well-regarded external packages** to delegate complex logic. A library with a clean, declarative API allows your application code to become a simple statement of intent.

When using an external package, ensure it meets these criteria:

  * **Good Reputation:** Widely used and respected in its ecosystem.
  * **Active Maintenance & Security:** Regularly updated, with a commitment from maintainers to address security issues.

### **Guideline for Scripting**

  * **Shell Script:** Use for simple, linear tasks: command pipelines, file operations. Leverage the declarative nature of tools like `find`, `grep`, `rsync`, and `xargs`.
  * **Python:** Default to Python for any script requiring complex logic, data structures, error handling, or API interactions. Use its rich ecosystem to keep the script clean and robust.

-----

### **Code Examples: The Hierarchy of Styles**

### **1. Good Declarative Code (The Goal)**

This is the ideal. The code is clean, expressive, and leverages powerful tools or features to state its intent clearly.

#### **Go: Simple HTTP Server**

*Library: Standard Library (`net/http`)*

```go
// GOOD: The standard library declaratively maps routes to handler functions.
package main
import ("fmt"; "log"; "net/http")

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, World!")
}

func main() {
	// Declaratively state that the "/hello" path is handled by helloHandler.
	http.HandleFunc("/hello", helloHandler)

	log.Println("Starting server on :8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatalf("Server failed to start: %v", err)
	}
}
```

#### **Python: Filesystem Operations**

*Library: Standard Library (`pathlib`)*

```python
# GOOD: An object-oriented, declarative way to handle filesystem paths.
from pathlib import Path

# Declaratively create a path and its parents, then write to a file.
p = Path("data/2025/reports")
p.mkdir(parents=True, exist_ok=True)
report_file = p / "january.txt"
report_file.write_text("This is the January report.")

print(f"File created: {report_file.exists()}")
print(f"Contents: {report_file.read_text()}")
```

#### **Node.js: Making API Calls**

*Library: [axios](https://github.com/axios/axios)*

```javascript
// GOOD: Declaratively describes an HTTP GET request.
const axios = require('axios');

async function getUser(id) {
  try {
    const response = await axios.get(`https://jsonplaceholder.typicode.com/users/${id}`);
    console.log('User found:', response.data.name);
  } catch (error) {
    console.error('Error fetching user:', error.message);
  }
}

getUser(1);
```

#### **Shell: Synchronizing Directories**

```shell
# GOOD: Declaratively defines the desired state of the destination directory.
# 'archive mode', 'verbose', 'compress', 'delete extraneous files'
rsync -avz --delete /path/to/source/ /path/to/destination/
```

-----

### **2. When Readability Trumps Pure Declarativeness (The Exception)**

This is the most nuanced category. In these cases, a direct, even slightly imperative, approach is **better** because the purely declarative version would be more confusing.

#### **Go: Complex Loop Logic**

```go
// READABLE > DECLARATIVE: A for loop is clearer than a complex filter predicate.
// Task: Find the first active admin, but log a warning for a legacy admin.
package main
import "fmt"
type User struct { Name string; IsActive bool; IsAdmin bool }

func findEligibleAdmin(users []User) *User {
	for i := range users {
		user := &users[i] // Use pointer to avoid copying
		if user.Name == "legacy_admin" {
			fmt.Println("Warning: Found legacy admin, skipping.")
			continue // Simple, direct control flow
		}
		if user.IsActive && user.IsAdmin {
			fmt.Println("Found eligible admin.")
			return user // Simple, direct exit
		}
	}
	return nil
}
// A declarative `funk.Find()` would require a single, complex lambda with side effects
// (the logging), making it less clean than this straightforward loop.
```

#### **Python: Multi-Step Data Transformation**

```python
# READABLE > DECLARATIVE: A loop with intermediate variables is easier to debug.
# Task: Process transactions, enriching and filtering at each step.
def process_transactions(transactions, user_db):
    processed = []
    for tx in transactions:
        # Step 1: Enrich with user data
        user_info = user_db.get(tx['user_id'])
        if not user_info:
            continue # Clear and simple skip

        tx_with_user = tx | {"user_name": user_info["name"]}

        # Step 2: Filter out small transactions
        if tx_with_user['amount'] < 10.00:
            continue

        # Step 3: Add a processing flag
        tx_with_user["processed"] = True
        processed.append(tx_with_user)

    return processed
# A single, long declarative chain of map/filter would be dense and hard to inspect.
# This imperative loop is clear, maintainable, and easy to debug step-by-step.
```

#### **Node.js: Complex Async Error Handling**

```javascript
// READABLE > DECLARATIVE: async/await with try/catch is clearer for complex error logic.
async function fetchDataWithRetry(url) {
  try {
    return await axios.get(url);
  } catch (error) {
    // A nested promise chain with .catch() would be much harder to read here.
    if (error.response && error.response.status === 401) {
      console.log('Token expired. Refreshing token...');
      // await refreshToken();
      // return await axios.get(url); // Retry logic
    } else if (error.response && error.response.status === 404) {
      console.log('Resource not found. Returning null.');
      return null;
    } else {
      // For all other errors, re-throw to be handled by the caller.
      throw new Error(`Unhandled network error: ${error.message}`);
    }
  }
}
```

#### **Shell: Conditional Scripting Based on Exit Codes**

```shell
# READABLE > DECLARATIVE: An if/elif/else structure is more explicit than a complex one-liner.
./my_backup_script.sh
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "Backup successful."
  ./run_cleanup.sh
elif [ $EXIT_CODE -eq 2 ]; then
  echo "Warning: Backup completed with minor, non-fatal errors."
else
  echo "CRITICAL: Backup failed with exit code $EXIT_CODE." >&2
  exit 1
fi
# Trying to achieve this with `&&` and `||` would be cryptic and error-prone.
```

-----

### **3. Imperative & Too Declarative Code (To Avoid)**

For completeness, you should continue to avoid these two anti-patterns as previously defined.

  * **Imperative Code:** The verbose, manual, step-by-step style that is hard to maintain (e.g., manual data validation loops).
  * **Too Declarative Code:** The esoteric style where an abstraction makes simple tasks more complex (e.g., using a reactive stream library for a simple array map, or point-free composition that obscures intent).
