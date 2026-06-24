# Regular Expressions

## Overview

C++11 introduced the `<regex>` library for pattern matching and text processing. **Important**: `std::regex` is notoriously slow compared to RE2, Hyperscan, or compile-time [ctre](https://github.com/hanickadot/ctre); pathological patterns can cause catastrophic backtracking or even stack overflow. Use it for moderate workloads; reach for alternatives in hot paths. This chapter covers ECMAScript syntax (the default grammar), matching, searching, replacing, and iterators.

```
┌────────────────────────────────────────────────────────┐
│              REGEX LIBRARY COMPONENTS                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  MATCHING       │  SEARCHING       │  REPLACING        │
│  ────────       │  ─────────       │  ─────────        │
│  • regex_match  │  • regex_search  │  • regex_replace  │
│  • Full match   │  • Find first    │  • Substitute     │
│  • Strict       │  • Sub-patterns  │  • Format         │
│                                                        │
│  ITERATORS      │  SYNTAX          │  FLAGS            │
│  ─────────      │  ──────          │  ─────            │
│  • sregex_iter  │  • ECMAScript    │  • icase          │
│  • token_iter   │  • basic         │  • multiline      │
│  • Find all     │  • extended      │  • optimize       │
│                 │  • grep, awk     │                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Regex Basics

### Creating Regex Objects

Default grammar is `std::regex::ECMAScript`. Use raw string literals `R"(...)"` to avoid double-escaping backslashes.

```cpp
#include <iostream>
#include <regex>

int main() {
    // Basic regex
    std::regex pattern("hello");
    
    // With flags
    std::regex case_insensitive("HELLO", std::regex::icase);
    
    // Different grammar — note the metacharacters differ per grammar!
    std::regex ecma_pattern("\\d+", std::regex::ECMAScript);  // Default; \d works
    // POSIX BRE/ERE do NOT support \d; use [0-9]. In BRE, + is literal unless \+,
    // so use [0-9][0-9]* for "one or more digits".
    std::regex basic_pattern("[0-9][0-9]*", std::regex::basic);
    std::regex extended_pattern("[0-9]+", std::regex::extended);
    
    // Optimize for repeated use
    std::regex optimized("pattern", std::regex::optimize);
    
    // Raw string literals (avoid escaping backslashes)
    std::regex email(R"(\w+@\w+\.\w+)");  // No need for \\w
    
    return 0;
}
```

### Common Regex Patterns
```
┌────────────────────────────────────────────────────────┐
│              Common Regex Patterns                     │
├────────────────────────────────────────────────────────┤
│ Pattern       │ Matches                                │
├───────────────┼────────────────────────────────────────┤
│ .             │ Any character except newline           │
│ \d            │ Digit [0-9]                            │
│ \w            │ Word character [A-Za-z0-9_]            │
│ \s            │ Whitespace [ \t\n\r\f\v]               │
│ \D \W \S      │ Negated versions                       │
│ ^             │ Start of string/line                   │
│ $             │ End of string/line                     │
│ [abc]         │ Character class (a, b, or c)           │
│ [^abc]        │ Negated class (not a, b, or c)         │
│ (abc)         │ Capture group                          │
│ (?:abc)       │ Non-capturing group                    │
│ a|b           │ Alternation (a or b)                   │
│ *             │ 0 or more (greedy)                     │
│ +             │ 1 or more (greedy)                     │
│ ?             │ 0 or 1 (greedy)                        │
│ {n}           │ Exactly n                              │
│ {n,}          │ n or more                              │
│ {n,m}         │ Between n and m                        │
│ *? +? ??      │ Non-greedy versions                    │
└────────────────────────────────────────────────────────┘
```

---

## regex_match - Full String Matching

`std::regex_match` requires the **entire** input to match. Use for validation; use `std::regex_search` to find a pattern anywhere in a string. Results land in `std::smatch` (or `std::cmatch` for C-strings).

### Basic Matching
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "hello123";
    
    // Check if entire string matches
    if (std::regex_match(text, std::regex("hello\\d+"))) {
        std::cout << "Matched!\n";
    }
    
    // Case insensitive
    if (std::regex_match("HELLO", std::regex("hello", std::regex::icase))) {
        std::cout << "Case insensitive match!\n";
    }
    
    // Must match entire string
    if (!std::regex_match("hello123world", std::regex("hello\\d+"))) {
        std::cout << "Doesn't match (extra 'world')\n";
    }
    
    return 0;
}
```

### Capturing Groups
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "2024-12-04";
    std::regex pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    std::smatch matches;
    
    if (std::regex_match(text, matches, pattern)) {
        std::cout << "Full match: " << matches[0] << '\n';  // 2024-12-04
        std::cout << "Year: " << matches[1] << '\n';        // 2024
        std::cout << "Month: " << matches[2] << '\n';       // 12
        std::cout << "Day: " << matches[3] << '\n';         // 04
    }
    
    return 0;
}
```

### Named Captures (via numbered)
```cpp
#include <regex>
#include <string>

// C++ doesn't have named groups, but you can document them
int main() {
    std::string email = "user@example.com";
    std::regex pattern(R"((\w+)@(\w+)\.(\w+))");
    std::smatch matches;
    
    if (std::regex_match(email, matches, pattern)) {
        // Document which index is which
        const int USER = 1;
        const int DOMAIN = 2;
        const int TLD = 3;
        
        std::cout << "User: " << matches[USER] << '\n';
        std::cout << "Domain: " << matches[DOMAIN] << '\n';
        std::cout << "TLD: " << matches[TLD] << '\n';
    }
    
    return 0;
}
```

---

## regex_search - Finding Patterns

`std::regex_search` finds the first match anywhere in the input. Use `std::sregex_iterator` / `std::cregex_iterator` to enumerate all matches.

### Basic Search
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "Contact us at support@example.com or sales@example.com";
    std::regex pattern(R"(\w+@\w+\.\w+)");
    std::smatch match;
    
    // Find first occurrence
    if (std::regex_search(text, match, pattern)) {
        std::cout << "Found email: " << match[0] << '\n';
        std::cout << "Position: " << match.position() << '\n';
        std::cout << "Length: " << match.length() << '\n';
        
        // Text before and after match
        std::cout << "Prefix: " << match.prefix() << '\n';
        std::cout << "Suffix: " << match.suffix() << '\n';
    }
    
    return 0;
}
```

### Finding All Matches
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "Prices: $10, $20.50, $3.99";
    std::regex pattern(R"(\$\d+(?:\.\d{2})?)");
    
    // Method 1: Using regex_search in loop
    std::smatch match;
    std::string::const_iterator search_start(text.cbegin());
    
    while (std::regex_search(search_start, text.cend(), match, pattern)) {
        std::cout << "Found: " << match[0] << '\n';
        search_start = match.suffix().first;
    }
    
    // Method 2: Using regex_iterator (better)
    auto begin = std::sregex_iterator(text.begin(), text.end(), pattern);
    auto end = std::sregex_iterator();
    
    for (auto it = begin; it != end; ++it) {
        std::cout << "Found: " << it->str() << '\n';
    }
    
    return 0;
}
```

---

## regex_replace - Substitution

Replacement strings use `$1`, `$2`, … for capture groups (`$$` for a literal `$`).

### Basic Replacement
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "Hello, World!";
    std::regex pattern("World");
    
    // Replace first occurrence
    std::string result = std::regex_replace(text, pattern, "Universe");
    std::cout << result << '\n';  // "Hello, Universe!"
    
    // Replace all occurrences
    std::string numbers = "1, 2, 3, 4, 5";
    std::string replaced = std::regex_replace(numbers, std::regex("\\d"), "X");
    std::cout << replaced << '\n';  // "X, X, X, X, X"
    
    return 0;
}
```

### Using Capture Groups in Replacement
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "2024-12-04";
    std::regex pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    
    // Rearrange date format: YYYY-MM-DD -> DD/MM/YYYY
    std::string result = std::regex_replace(text, pattern, "$3/$2/$1");
    std::cout << result << '\n';  // "04/12/2024"
    
    // Swap first and last name
    std::string name = "Doe, John";
    std::string swapped = std::regex_replace(
        name,
        std::regex(R"((\w+), (\w+))"),
        "$2 $1"
    );
    std::cout << swapped << '\n';  // "John Doe"
    
    return 0;
}
```

### Replacement Flags
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "foo FOO Foo";
    std::regex pattern("foo", std::regex::icase);
    
    // Default: Replace all
    std::cout << std::regex_replace(text, pattern, "bar") << '\n';
    // "bar bar bar"
    
    // Replace only first
    std::cout << std::regex_replace(
        text, pattern, "bar",
        std::regex_constants::format_first_only
    ) << '\n';
    // "bar FOO Foo"
    
    // Don't copy unmatched parts
    std::cout << std::regex_replace(
        text, pattern, "bar",
        std::regex_constants::format_no_copy
    ) << '\n';
    // "barbarbar" (only replacements)
    
    return 0;
}
```

---

## Regex Iterators

### sregex_iterator
```cpp
#include <regex>
#include <string>
#include <iostream>

int main() {
    std::string text = "Error on line 10, warning on line 25, error on line 42";
    std::regex pattern(R"((error|warning) on line (\d+))");
    
    auto begin = std::sregex_iterator(text.begin(), text.end(), pattern);
    auto end = std::sregex_iterator();
    
    for (auto it = begin; it != end; ++it) {
        std::smatch match = *it;
        std::cout << "Match: " << match.str() << '\n';
        std::cout << "  Type: " << match[1] << '\n';
        std::cout << "  Line: " << match[2] << '\n';
    }
    
    return 0;
}
```

### sregex_token_iterator
```cpp
#include <regex>
#include <string>
#include <iostream>
#include <vector>

int main() {
    std::string text = "one,two,three,four";
    std::regex delim(",");
    
    // Split by delimiter
    std::sregex_token_iterator begin(text.begin(), text.end(), delim, -1);
    std::sregex_token_iterator end;
    
    std::vector<std::string> tokens(begin, end);
    for (const auto& token : tokens) {
        std::cout << token << '\n';
    }
    
    // Extract only the delimiters
    std::sregex_token_iterator delim_begin(text.begin(), text.end(), delim, 0);
    for (auto it = delim_begin; it != end; ++it) {
        std::cout << "Delimiter: " << *it << '\n';
    }
    
    // Extract specific groups
    std::string dates = "2024-12-04, 2023-11-03";
    std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    
    // Extract only years (group 1)
    std::sregex_token_iterator year_it(dates.begin(), dates.end(), date_pattern, 1);
    for (auto it = year_it; it != end; ++it) {
        std::cout << "Year: " << *it << '\n';
    }
    
    return 0;
}
```

---

## Common Use Cases

### Email Validation
```cpp
#include <regex>
#include <string>

bool is_valid_email(const std::string& email) {
    // Simple email pattern
    std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
    return std::regex_match(email, pattern);
}

int main() {
    std::cout << is_valid_email("user@example.com") << '\n';  // 1 (true)
    std::cout << is_valid_email("invalid.email") << '\n';     // 0 (false)
    
    return 0;
}
```

### URL Parsing
```cpp
#include <regex>
#include <string>
#include <iostream>

struct URL {
    std::string protocol;
    std::string domain;
    std::string port;
    std::string path;
    std::string query;
};

URL parse_url(const std::string& url) {
    std::regex pattern(
        R"(^(https?):\/\/([^:/]+)(?::(\d+))?(\/[^?]*)?\??(.*)$)"
    );
    std::smatch matches;
    
    URL result;
    if (std::regex_match(url, matches, pattern)) {
        result.protocol = matches[1];
        result.domain = matches[2];
        result.port = matches[3];
        result.path = matches[4];
        result.query = matches[5];
    }
    
    return result;
}

int main() {
    URL url = parse_url("https://example.com:8080/path/to/page?key=value");
    
    std::cout << "Protocol: " << url.protocol << '\n';
    std::cout << "Domain: " << url.domain << '\n';
    std::cout << "Port: " << url.port << '\n';
    std::cout << "Path: " << url.path << '\n';
    std::cout << "Query: " << url.query << '\n';
    
    return 0;
}
```

### Log Parsing
```cpp
#include <regex>
#include <string>
#include <fstream>
#include <iostream>

struct LogEntry {
    std::string timestamp;
    std::string level;
    std::string message;
};

std::vector<LogEntry> parse_log(const std::string& log_text) {
    std::vector<LogEntry> entries;
    
    // Pattern: [YYYY-MM-DD HH:MM:SS] LEVEL: message
    std::regex pattern(R"(\[([\d\-: ]+)\] (\w+): (.+))");
    
    auto begin = std::sregex_iterator(log_text.begin(), log_text.end(), pattern);
    auto end = std::sregex_iterator();
    
    for (auto it = begin; it != end; ++it) {
        LogEntry entry;
        entry.timestamp = (*it)[1];
        entry.level = (*it)[2];
        entry.message = (*it)[3];
        entries.push_back(entry);
    }
    
    return entries;
}
```

### Phone Number Formatting
```cpp
#include <regex>
#include <string>
#include <iostream>

std::string format_phone_number(const std::string& phone) {
    // Remove all non-digit characters
    std::string digits = std::regex_replace(phone, std::regex(R"(\D)"), "");
    
    // Format as (XXX) XXX-XXXX
    std::regex pattern(R"((\d{3})(\d{3})(\d{4}))");
    return std::regex_replace(digits, pattern, "($1) $2-$3");
}

int main() {
    std::cout << format_phone_number("1234567890") << '\n';
    // (123) 456-7890
    
    std::cout << format_phone_number("123-456-7890") << '\n';
    // (123) 456-7890
    
    std::cout << format_phone_number("(123) 456-7890") << '\n';
    // (123) 456-7890
    
    return 0;
}
```

---

## Performance Considerations

`std::regex` compiles patterns into a backtracking engine — compilation is expensive, matching can be very slow, and nested quantifiers like `(a+)+b` risk exponential time or stack overflow on long inputs.

```
┌────────────────────────────────────────────────────────┐
│              Regex Performance Tips                    │
├────────────────────────────────────────────────────────┤
│                                                        │
│ 1. Compile once, use many times (store std::regex)     │
│ 2. std::regex::optimize — slower compile, faster match │
│ 3. Avoid catastrophic backtracking: (a+)+ → a+         │
│ 4. Use (?:…) non-capturing groups when possible        │
│ 5. For full-string checks, anchor: ^pattern$           │
│ 6. Simple substring? Use string::find / starts_with    │
│ 7. Hot path? Consider RE2, ctre, or a hand parser      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

Benchmark with [steady_clock](18_time_chrono.md):

### Benchmarking Example
```cpp
#include <chrono>
#include <iostream>
#include <regex>
#include <string>

void benchmark_regex() {
    using namespace std::chrono;
    
    std::string text = "test@example.com";
    
    // Compile every time (SLOW)
    {
        auto start = steady_clock::now();
        for (int i = 0; i < 10000; ++i) {
            std::regex r(R"(\w+@\w+\.\w+)");
            std::regex_match(text, r);
        }
        auto elapsed = duration_cast<milliseconds>(steady_clock::now() - start);
        std::cout << "Compile each time: " << elapsed.count() << "ms\n";
    }
    
    // Compile once (FAST)
    {
        std::regex r(R"(\w+@\w+\.\w+)", std::regex::optimize);
        auto start = steady_clock::now();
        for (int i = 0; i < 10000; ++i) {
            std::regex_match(text, r);
        }
        auto elapsed = duration_cast<milliseconds>(steady_clock::now() - start);
        std::cout << "Compile once: " << elapsed.count() << "ms\n";
    }
}
```

---

## Error Handling

### Catching Regex Errors
```cpp
#include <regex>
#include <iostream>

int main() {
    try {
        // Invalid regex: unmatched parenthesis
        std::regex invalid("(abc");
    }
    catch (const std::regex_error& e) {
        std::cout << "Regex error: " << e.what() << '\n';
        std::cout << "Error code: " << e.code() << '\n';
    }
    
    // Check error codes
    try {
        std::regex r("[z-a]");  // Invalid range
    }
    catch (const std::regex_error& e) {
        if (e.code() == std::regex_constants::error_range) {
            std::cout << "Invalid character range\n";
        }
    }
    
    return 0;
}
```

---

## Best Practices

### 1. Use Raw String Literals
```cpp
// GOOD: Clear, no double backslashes
std::regex pattern(R"(\d+\.\d+)");

// BAD: Hard to read
std::regex pattern("\\d+\\.\\d+");
```

### 2. Document Complex Patterns
```cpp
// GOOD: Documented regex
// Pattern: (year)-(month)-(day)
// Example: 2024-12-04
std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");

const int YEAR = 1;
const int MONTH = 2;
const int DAY = 3;
```

### 3. Validate Input Length
```cpp
bool is_safe_input(const std::string& input) {
    if (input.size() > 10000) return false;
    return true;
}
```

### 4. Use Appropriate Matching Function
```cpp
// regex_match: Entire string must match
// Use for: Validation
if (std::regex_match(email, email_pattern)) { /* valid email */ }

// regex_search: Find pattern anywhere
// Use for: Extraction
if (std::regex_search(text, match, pattern)) { /* found */ }

// regex_replace: Replace occurrences
// Use for: Transformation
auto result = std::regex_replace(text, pattern, replacement);
```

---

## Common Pitfalls

### 1. Forgetting to Escape Special Characters
```cpp
// BAD: . matches any character
std::regex pattern("example.com");  // Matches "exampleXcom"

// GOOD: \. matches literal dot
std::regex pattern(R"(example\.com)");
```

### 2. Catastrophic Backtracking and Stack Overflow
```cpp
// BAD: exponential backtracking; may hang or overflow stack
std::regex bad(R"((a+)+b)");
std::string input(30, 'a');
std::regex_search(input, bad);  // painfully slow

// GOOD: linear
std::regex good(R"(a+b)");
```

### 3. Not Using std::regex::optimize
```cpp
// If using regex repeatedly
std::regex pattern("\\w+", std::regex::optimize);
```

---

## Alternatives to std::regex

When `std::regex` is too slow or unsafe:

| Library | Notes |
|---------|-------|
| [RE2](https://github.com/google/re2) | Linear-time, no backtracking; widely used in production |
| [ctre](https://github.com/hanickadot/ctre) | Compile-time regex; zero runtime compilation |
| `string::find` / `starts_with` | [Modern C++ Features](07_modern_features.md) for simple cases |
| Dedicated parser | Log/CSV/binary formats — often clearer than regex |

```cpp
// Simple substring search
if (text.find("substring") != std::string::npos) { /* ... */ }

if (text.starts_with("prefix")) { /* C++20 */ }
```

---

## Complete Practical Example: Data Validator and Parser

Here's a comprehensive example integrating regex matching, searching, replacing, and validation:

```cpp
#include <iostream>
#include <regex>
#include <string>
#include <vector>
#include <map>
#include <fstream>
#include <sstream>

// Data validation class using multiple regex patterns
class DataValidator {
private:
    // Compiled regex patterns (compile once, use many times)
    std::regex email_pattern_{R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})"};
    std::regex phone_pattern_{R"(\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4})"};
    std::regex url_pattern_{R"(https?://[^\s]+)"};
    std::regex ip_pattern_{R"(\b(?:\d{1,3}\.){3}\d{1,3}\b)"};
    std::regex date_pattern_{R"(\d{4}-\d{2}-\d{2})"};
    std::regex credit_card_pattern_{R"(\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4})"};
    std::regex ssn_pattern_{R"(\d{3}-\d{2}-\d{4})"};
    
public:
    // Email validation
    bool is_valid_email(const std::string& email) const {
        return std::regex_match(email, email_pattern_);
    }
    
    // Extract all emails from text
    std::vector<std::string> extract_emails(const std::string& text) const {
        std::vector<std::string> emails;
        
        auto begin = std::sregex_iterator(text.begin(), text.end(), email_pattern_);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            emails.push_back(it->str());
        }
        
        return emails;
    }
    
    // Phone number validation and formatting
    struct PhoneNumber {
        std::string original;
        std::string formatted;
        bool valid;
    };
    
    PhoneNumber validate_phone(const std::string& phone) const {
        PhoneNumber result{phone, "", false};
        
        if (std::regex_match(phone, phone_pattern_)) {
            result.valid = true;
            
            // Extract digits only
            std::string digits = std::regex_replace(phone, 
                                                    std::regex(R"(\D)"), "");
            
            // Format as (XXX) XXX-XXXX
            if (digits.length() == 10) {
                result.formatted = "(" + digits.substr(0, 3) + ") " +
                                  digits.substr(3, 3) + "-" +
                                  digits.substr(6, 4);
            }
        }
        
        return result;
    }
    
    // URL extraction and validation
    std::vector<std::string> extract_urls(const std::string& text) const {
        std::vector<std::string> urls;
        
        auto begin = std::sregex_iterator(text.begin(), text.end(), url_pattern_);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            urls.push_back(it->str());
        }
        
        return urls;
    }
    
    // Validate and parse date
    struct Date {
        int year, month, day;
        bool valid;
    };
    
    Date parse_date(const std::string& date_str) const {
        std::smatch matches;
        Date result{0, 0, 0, false};
        
        if (std::regex_match(date_str, matches, date_pattern_)) {
            try {
                result.year = std::stoi(date_str.substr(0, 4));
                result.month = std::stoi(date_str.substr(5, 2));
                result.day = std::stoi(date_str.substr(8, 2));
                
                // Basic validation
                if (result.year >= 1900 && result.year <= 2100 &&
                    result.month >= 1 && result.month <= 12 &&
                    result.day >= 1 && result.day <= 31) {
                    result.valid = true;
                }
            } catch (...) {
                result.valid = false;
            }
        }
        
        return result;
    }
    
    // Mask sensitive data (credit cards, SSN)
    std::string mask_credit_cards(const std::string& text) const {
        return std::regex_replace(text, credit_card_pattern_,
            [](const std::smatch& match) {
                std::string card = match.str();
                // Remove all non-digits
                card = std::regex_replace(card, std::regex(R"(\D)"), "");
                // Mask all but last 4 digits
                if (card.length() >= 4) {
                    return "**** **** **** " + card.substr(card.length() - 4);
                }
                return std::string("****");
            });
    }
    
    std::string mask_ssn(const std::string& text) const {
        return std::regex_replace(text, ssn_pattern_, "***-**-****");
    }
    
    // Find IP addresses
    std::vector<std::string> extract_ip_addresses(const std::string& text) const {
        std::vector<std::string> ips;
        
        auto begin = std::sregex_iterator(text.begin(), text.end(), ip_pattern_);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            ips.push_back(it->str());
        }
        
        return ips;
    }
};

// Log parser using regex
class LogParser {
private:
    // Pattern: [TIMESTAMP] [LEVEL] [SOURCE] Message
    std::regex log_pattern_{R"(\[([^\]]+)\]\s*\[(\w+)\]\s*\[([^\]]+)\]\s*(.+))"};
    
public:
    struct LogEntry {
        std::string timestamp;
        std::string level;
        std::string source;
        std::string message;
    };
    
    bool parse_log_line(const std::string& line, LogEntry& entry) const {
        std::smatch matches;
        
        if (std::regex_match(line, matches, log_pattern_)) {
            entry.timestamp = matches[1];
            entry.level = matches[2];
            entry.source = matches[3];
            entry.message = matches[4];
            return true;
        }
        
        return false;
    }
    
    std::vector<LogEntry> parse_log_file(const std::string& filename) const {
        std::vector<LogEntry> entries;
        std::ifstream file(filename);
        std::string line;
        
        while (std::getline(file, line)) {
            LogEntry entry;
            if (parse_log_line(line, entry)) {
                entries.push_back(entry);
            }
        }
        
        return entries;
    }
    
    // Filter logs by level
    std::vector<LogEntry> filter_by_level(
        const std::vector<LogEntry>& entries,
        const std::string& level) const {
        
        std::vector<LogEntry> filtered;
        
        for (const auto& entry : entries) {
            if (entry.level == level) {
                filtered.push_back(entry);
            }
        }
        
        return filtered;
    }
};

// Text sanitizer/cleaner
class TextSanitizer {
public:
    // Remove HTML tags
    std::string remove_html_tags(const std::string& text) const {
        return std::regex_replace(text, std::regex(R"(<[^>]*>)"), "");
    }
    
    // Normalize whitespace
    std::string normalize_whitespace(const std::string& text) const {
        // Replace multiple spaces with single space
        std::string result = std::regex_replace(text, 
                                               std::regex(R"(\s+)"), " ");
        
        // Trim leading/trailing spaces
        result = std::regex_replace(result, std::regex(R"(^\s+|\s+$)"), "");
        
        return result;
    }
    
    // Extract hashtags
    std::vector<std::string> extract_hashtags(const std::string& text) const {
        std::vector<std::string> hashtags;
        std::regex pattern(R"(#(\w+))");
        
        auto begin = std::sregex_iterator(text.begin(), text.end(), pattern);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            hashtags.push_back((*it)[1].str());  // Group 1 is the tag without #
        }
        
        return hashtags;
    }
    
    // Extract mentions
    std::vector<std::string> extract_mentions(const std::string& text) const {
        std::vector<std::string> mentions;
        std::regex pattern(R"(@(\w+))");
        
        auto begin = std::sregex_iterator(text.begin(), text.end(), pattern);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            mentions.push_back((*it)[1].str());
        }
        
        return mentions;
    }
    
    // Convert URLs to links
    std::string linkify_urls(const std::string& text) const {
        std::regex url_pattern(R"((https?://[^\s]+))");
        return std::regex_replace(text, url_pattern, "<a href=\"$1\">$1</a>");
    }
};

// CSV parser using regex
class CSVParser {
private:
    std::regex csv_pattern_{R"((?:^|,)(?:"([^"]*)"|([^,]*)))"};
    
public:
    std::vector<std::string> parse_line(const std::string& line) const {
        std::vector<std::string> fields;
        
        auto begin = std::sregex_iterator(line.begin(), line.end(), csv_pattern_);
        auto end = std::sregex_iterator();
        
        for (auto it = begin; it != end; ++it) {
            // Get either quoted (group 1) or unquoted (group 2) field
            std::string field = (*it)[1].matched ? (*it)[1].str() : (*it)[2].str();
            fields.push_back(field);
        }
        
        return fields;
    }
};

int main() {
    std::cout << "=== Data Validator and Parser Demo ===\n\n";
    
    // 1. Email validation and extraction
    std::cout << "--- Email Validation ---\n";
    DataValidator validator;
    
    std::string text1 = "Contact us at info@example.com or support@test.org";
    
    auto emails = validator.extract_emails(text1);
    std::cout << "Found " << emails.size() << " emails:\n";
    for (const auto& email : emails) {
        std::cout << "  " << email 
                  << " (" << (validator.is_valid_email(email) ? "valid" : "invalid") 
                  << ")\n";
    }
    
    // 2. Phone number validation and formatting
    std::cout << "\n--- Phone Number Validation ---\n";
    std::vector<std::string> phones = {
        "555-123-4567",
        "(555) 123-4567",
        "5551234567",
        "invalid"
    };
    
    for (const auto& phone : phones) {
        auto result = validator.validate_phone(phone);
        std::cout << phone << " -> ";
        if (result.valid) {
            std::cout << result.formatted << " (valid)\n";
        } else {
            std::cout << "invalid\n";
        }
    }
    
    // 3. URL extraction
    std::cout << "\n--- URL Extraction ---\n";
    std::string text2 = "Visit https://example.com or http://test.org for more info";
    
    auto urls = validator.extract_urls(text2);
    std::cout << "Found " << urls.size() << " URLs:\n";
    for (const auto& url : urls) {
        std::cout << "  " << url << "\n";
    }
    
    // 4. Date parsing
    std::cout << "\n--- Date Parsing ---\n";
    std::vector<std::string> dates = {"2024-12-04", "2024-13-01", "invalid"};
    
    for (const auto& date_str : dates) {
        auto date = validator.parse_date(date_str);
        std::cout << date_str << " -> ";
        if (date.valid) {
            std::cout << "Year: " << date.year << ", Month: " << date.month 
                      << ", Day: " << date.day << "\n";
        } else {
            std::cout << "invalid\n";
        }
    }
    
    // 5. Masking sensitive data
    std::cout << "\n--- Data Masking ---\n";
    std::string sensitive = "Card: 1234-5678-9012-3456, SSN: 123-45-6789";
    std::cout << "Original: " << sensitive << "\n";
    std::cout << "Masked:   " << validator.mask_credit_cards(sensitive) << "\n";
    
    // 6. Text sanitization
    std::cout << "\n--- Text Sanitization ---\n";
    TextSanitizer sanitizer;
    
    std::string html = "<p>Hello <b>World</b>!</p>";
    std::cout << "HTML: " << html << "\n";
    std::cout << "Clean: " << sanitizer.remove_html_tags(html) << "\n";
    
    std::string messy = "  Too    many     spaces  ";
    std::cout << "Messy: '" << messy << "'\n";
    std::cout << "Clean: '" << sanitizer.normalize_whitespace(messy) << "'\n";
    
    // 7. Social media parsing
    std::cout << "\n--- Social Media Parsing ---\n";
    std::string tweet = "Check out #cpp #programming! Thanks @john and @jane";
    
    auto hashtags = sanitizer.extract_hashtags(tweet);
    std::cout << "Hashtags: ";
    for (const auto& tag : hashtags) {
        std::cout << "#" << tag << " ";
    }
    std::cout << "\n";
    
    auto mentions = sanitizer.extract_mentions(tweet);
    std::cout << "Mentions: ";
    for (const auto& mention : mentions) {
        std::cout << "@" << mention << " ";
    }
    std::cout << "\n";
    
    // 8. CSV parsing
    std::cout << "\n--- CSV Parsing ---\n";
    CSVParser csv_parser;
    
    std::string csv_line = R"(John,"Doe, Jr.",30,"New York")";
    auto fields = csv_parser.parse_line(csv_line);
    
    std::cout << "CSV line: " << csv_line << "\n";
    std::cout << "Parsed fields:\n";
    for (size_t i = 0; i < fields.size(); ++i) {
        std::cout << "  Field " << i << ": '" << fields[i] << "'\n";
    }
    
    std::cout << "\n=== Demo Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **regex_match / regex_search / regex_replace**: Core API
- **smatch / capture groups**: `match[i]` and `$1` back-references
- **sregex_iterator / sregex_token_iterator**: All matches and tokenization
- **Raw string literals**: Readable patterns
- **Compile once**: Member `std::regex` fields

---

## Related Topics

- [I/O & Filesystem](16_io_filesystem.md) — reading log files and text streams to parse
- [Algorithms](06_algorithms.md) — `std::search`, `std::find` for simpler matching
- [Modern C++ Features](07_modern_features.md) — `starts_with`, `ends_with`, `string_view`
- [Lambdas](10_lambdas.md) — `regex_replace` with formatter lambdas
- [Time and Chrono](18_time_chrono.md) — benchmarking regex vs alternatives
- [Best Practices](13_best_practices.md) — input validation and defensive parsing
- [Quick Reference](99_quick_reference.md) — regex API summary

## Next Steps
- **Next**: [Modules →](21_modules.md)
- **Previous**: [← Memory Management](19_memory_allocators.md)

---
*Chapter 20 — Regular Expressions*

