# I/O, Filesystem, and Formatting

## Overview

C++ provides comprehensive facilities for input/output, filesystem manipulation, and modern string formatting. For error handling during I/O, see [Exception Handling](17_exceptions.md); for time formatting in logs, see [Time and Chrono](18_time_chrono.md).

```
┌────────────────────────────────────────────────────────┐
│              I/O AND FILESYSTEM TOOLS                  │
├────────────────────────────────────────────────────────┤
│                                                        │
│  STREAMS        │  FILESYSTEM      │  FORMATTING       │
│  ───────        │  ──────────      │  ──────────       │
│  • iostream     │  • std::path     │  • std::format    │
│  • fstream      │  • directory     │  • printf-like    │
│  • sstream      │  • operations    │  • Type-safe      │
│  • manipulators │  • permissions   │  • Custom types   │
│                                                        │
│  C++11: Move semantics for streams                     │
│  C++17: std::filesystem                                │
│  C++20: std::format, std::span                         │
│  C++23: std::print, formatting improvements            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Stream Basics

### Standard Streams
```cpp
#include <iostream>

int main() {
    // Standard output stream
    std::cout << "Hello, World!" << std::endl;
    
    // Standard error stream (unbuffered)
    std::cerr << "Error message" << std::endl;
    
    // Standard log stream (buffered)
    std::clog << "Log message" << std::endl;
    
    // Standard input stream
    int number;
    std::cout << "Enter a number: ";
    std::cin >> number;
    
    return 0;
}
```

### Stream State Flags
```cpp
#include <iostream>
#include <limits>

int main() {
    int value;
    std::cin >> value;
    
    // Check stream state
    if (std::cin.good()) {
        std::cout << "Stream is good\n";
    }
    
    if (std::cin.fail()) {
        std::cout << "Input failed\n";
        std::cin.clear();  // Clear error flags
        std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    }
    
    if (std::cin.eof()) {
        std::cout << "End of file reached\n";
    }
    
    if (std::cin.bad()) {
        std::cout << "Fatal stream error\n";
    }
    
    return 0;
}
```

### Stream State Diagram
```
Stream States:
┌────────────────────────────────────────────────────────┐
│                                                        │
│  goodbit (0) ──► All OK, ready for I/O                 │
│  eofbit      ──► End-of-file reached                   │
│  failbit     ──► I/O operation failed (recoverable)    │
│  badbit      ──► Fatal error (stream corrupted)        │
│                                                        │
│  good() → returns !fail()                              │
│  eof()  → returns eofbit set                           │
│  fail() → returns failbit or badbit set                │
│  bad()  → returns badbit set                           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## File I/O

### Reading and Writing Files
```cpp
#include <fstream>
#include <iostream>
#include <string>

int main() {
    // Writing to file
    {
        std::ofstream out("output.txt");
        if (out.is_open()) {
            out << "Hello, File!" << std::endl;
            out << "Line 2" << std::endl;
            out << 42 << " " << 3.14 << std::endl;
        }
        // File automatically closed when out goes out of scope
    }
    
    // Reading from file
    {
        std::ifstream in("output.txt");
        if (in.is_open()) {
            std::string line;
            while (std::getline(in, line)) {
                std::cout << line << std::endl;
            }
        }
    }
    
    // Read/write mode
    {
        std::fstream file("data.txt", std::ios::in | std::ios::out);
        if (file.is_open()) {
            // Read and write operations
        }
    }
    
    return 0;
}
```

### File Open Modes
```cpp
#include <fstream>

int main() {
    // Open modes
    std::ofstream out1("file.txt", std::ios::out);      // Write (default)
    std::ofstream out2("file.txt", std::ios::app);      // Append
    std::ofstream out3("file.txt", std::ios::trunc);    // Truncate
    
    std::ifstream in("file.txt", std::ios::in);         // Read (default)
    
    std::fstream file1("file.txt", std::ios::in | std::ios::out);  // R/W
    std::fstream file2("file.txt", std::ios::binary);   // Binary mode
    std::fstream file3("file.txt", std::ios::ate);      // Start at end
    
    return 0;
}
```

### Binary File I/O

⚠️ **Gotcha**: Writing raw structs with `write()` is fragile — padding, endianness, and version changes break portability. Prefer explicit serialization or a format like JSON/Protobuf for durable storage.

```cpp
#include <fstream>
#include <vector>
#include <cstring>

struct Record {
    char name[50];
    int age;
    double salary;
};

int main() {
    // Write binary data
    {
        std::ofstream out("data.bin", std::ios::binary);
        
        Record rec;
        std::strcpy(rec.name, "John Doe");
        rec.age = 30;
        rec.salary = 50000.0;
        
        out.write(reinterpret_cast<const char*>(&rec), sizeof(Record));
    }
    
    // Read binary data
    {
        std::ifstream in("data.bin", std::ios::binary);
        
        Record rec;
        in.read(reinterpret_cast<char*>(&rec), sizeof(Record));
        
        std::cout << "Name: " << rec.name << std::endl;
        std::cout << "Age: " << rec.age << std::endl;
        std::cout << "Salary: " << rec.salary << std::endl;
    }
    
    // Write array
    {
        std::vector<int> data = {1, 2, 3, 4, 5};
        std::ofstream out("array.bin", std::ios::binary);
        out.write(reinterpret_cast<const char*>(data.data()), 
                  data.size() * sizeof(int));
    }
    
    return 0;
}
```

### File Position Operations
```cpp
#include <fstream>

int main() {
    std::fstream file("data.txt", std::ios::in | std::ios::out);
    
    // Get current position
    std::streampos pos = file.tellg();  // Input position
    std::streampos pos2 = file.tellp(); // Output position
    
    // Seek to position
    file.seekg(0, std::ios::beg);     // Beginning
    file.seekg(0, std::ios::end);     // End
    file.seekg(10, std::ios::cur);    // 10 bytes forward from current
    
    // Get file size
    file.seekg(0, std::ios::end);
    std::streamsize size = file.tellg();
    file.seekg(0, std::ios::beg);
    
    return 0;
}
```

---

## String Streams

### std::stringstream
```cpp
#include <sstream>
#include <string>

int main() {
    // Output string stream
    std::ostringstream oss;
    oss << "Number: " << 42 << ", Pi: " << 3.14;
    std::string result = oss.str();
    
    // Input string stream
    std::istringstream iss("42 3.14 hello");
    int num;
    double pi;
    std::string word;
    iss >> num >> pi >> word;
    
    // Bidirectional string stream
    std::stringstream ss;
    ss << "Value: " << 100;
    std::string str = ss.str();
    
    // Clear and reuse
    ss.str("");  // Clear contents
    ss.clear();  // Clear state flags
    ss << "New value";
    
    return 0;
}
```

### String Stream Use Cases
```cpp
#include <sstream>
#include <vector>

// Convert to string
template<typename T>
std::string to_string_custom(const T& value) {
    std::ostringstream oss;
    oss << value;
    return oss.str();
}

// Parse CSV line
std::vector<std::string> parse_csv(const std::string& line) {
    std::vector<std::string> result;
    std::istringstream iss(line);
    std::string field;
    
    while (std::getline(iss, field, ',')) {
        result.push_back(field);
    }
    
    return result;
}

// Concatenate with formatting
std::string build_message(int code, const std::string& msg) {
    std::ostringstream oss;
    oss << "[Error " << code << "] " << msg;
    return oss.str();
}
```

---

## Stream Manipulators

### Standard Manipulators
```cpp
#include <iostream>
#include <iomanip>

int main() {
    // Boolean formatting
    std::cout << std::boolalpha << true << std::endl;  // "true"
    std::cout << std::noboolalpha << true << std::endl; // "1"
    
    // Integer base
    std::cout << std::dec << 42 << std::endl;    // Decimal: 42
    std::cout << std::hex << 42 << std::endl;    // Hex: 2a
    std::cout << std::oct << 42 << std::endl;    // Octal: 52
    
    // Show base prefix
    std::cout << std::showbase << std::hex << 42 << std::endl;  // 0x2a
    
    // Floating point
    std::cout << std::fixed << 3.14159 << std::endl;      // Fixed notation
    std::cout << std::scientific << 3.14159 << std::endl; // Scientific
    std::cout << std::defaultfloat << 3.14159 << std::endl;
    
    // Precision
    std::cout << std::setprecision(2) << 3.14159 << std::endl;  // 3.14
    
    // Width and fill
    std::cout << std::setw(10) << 42 << std::endl;           // "        42"
    std::cout << std::setfill('0') << std::setw(5) << 42 << std::endl;  // "00042"
    
    // Alignment
    std::cout << std::left << std::setw(10) << "left" << std::endl;
    std::cout << std::right << std::setw(10) << "right" << std::endl;
    
    return 0;
}
```

### Creating Custom Manipulators
```cpp
#include <iostream>

// Simple manipulator (no parameters)
std::ostream& bold(std::ostream& os) {
    return os << "\033[1m";
}

std::ostream& reset(std::ostream& os) {
    return os << "\033[0m";
}

// Manipulator with parameters
struct color {
    int code;
    color(int c) : code(c) {}
    
    friend std::ostream& operator<<(std::ostream& os, const color& c) {
        return os << "\033[38;5;" << c.code << "m";
    }
};

int main() {
    std::cout << bold << "Bold text" << reset << std::endl;
    std::cout << color(196) << "Red text" << reset << std::endl;
    
    return 0;
}
```

---

## std::filesystem (C++17)

`std::filesystem` operations have two overload styles: **throwing** (default, throws `std::filesystem::filesystem_error`) and **non-throwing** (`std::error_code&` out-parameter). Prefer the `error_code` overload in library code and hot paths where you handle failures explicitly — see [Exception Handling](17_exceptions.md) for `std::error_code` vs exceptions.

### Path Operations
```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    // Create path
    fs::path p1 = "/usr/local/bin";
    fs::path p2 = "C:\\Users\\Name\\Documents";
    fs::path p3 = p1 / "program";  // Path concatenation
    
    // Path components
    fs::path file = "/home/user/data/file.txt";
    std::cout << "Root: " << file.root_name() << std::endl;
    std::cout << "Parent: " << file.parent_path() << std::endl;
    std::cout << "Filename: " << file.filename() << std::endl;
    std::cout << "Stem: " << file.stem() << std::endl;          // "file"
    std::cout << "Extension: " << file.extension() << std::endl; // ".txt"
    
    // Path manipulation
    fs::path p = "/home/user";
    p /= "documents";           // Append
    p.replace_filename("new.txt");
    p.replace_extension(".dat");
    
    // Path queries
    if (file.has_filename()) { /* ... */ }
    if (file.has_extension()) { /* ... */ }
    if (file.is_absolute()) { /* ... */ }
    if (file.is_relative()) { /* ... */ }
    
    return 0;
}
```

### File Operations
```cpp
#include <filesystem>
#include <fstream>
#include <system_error>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    // Non-throwing overload (preferred in robust code)
    std::error_code ec;
    if (!fs::exists("file.txt", ec)) {
        if (ec) std::cerr << ec.message() << '\n';
        else std::cout << "File does not exist\n";
    }
    
    // Throwing overload — convenient in scripts, risky in libraries
    if (fs::exists("file.txt")) {
        std::cout << "File exists\n";
    }
    
    // File size (throws if missing)
    uintmax_t size = fs::file_size("file.txt");
    std::cout << "Size: " << size << " bytes\n";
    
    // File type
    if (fs::is_regular_file("file.txt")) { /* ... */ }
    if (fs::is_directory("folder")) { /* ... */ }
    if (fs::is_symlink("link")) { /* ... */ }
    
    // Copy file
    fs::copy("source.txt", "dest.txt");
    fs::copy("source.txt", "dest.txt", fs::copy_options::overwrite_existing);
    
    // Rename/move
    fs::rename("old.txt", "new.txt");
    
    // Remove
    fs::remove("file.txt");              // Remove file
    fs::remove_all("directory");         // Remove directory recursively
    
    // Create directory
    fs::create_directory("new_folder");
    fs::create_directories("path/to/nested/folder");
    
    // Current path
    fs::path current = fs::current_path();
    fs::current_path("/tmp");            // Change current directory
    
    return 0;
}
```

### Directory Iteration
```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    // Iterate directory (non-recursive)
    for (const auto& entry : fs::directory_iterator(".")) {
        std::cout << entry.path() << std::endl;
        
        if (entry.is_regular_file()) {
            std::cout << "  File, size: " << entry.file_size() << std::endl;
        } else if (entry.is_directory()) {
            std::cout << "  Directory" << std::endl;
        }
    }
    
    // Recursive iteration
    for (const auto& entry : fs::recursive_directory_iterator(".")) {
        std::cout << entry.path() << " (depth: " 
                  << entry.depth() << ")" << std::endl;
    }
    
    // Filtered iteration
    for (const auto& entry : fs::directory_iterator(".")) {
        if (entry.path().extension() == ".txt") {
            std::cout << "Text file: " << entry.path() << std::endl;
        }
    }
    
    return 0;
}
```

### File Permissions
```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
    fs::path file = "example.txt";
    
    // Get permissions
    fs::perms p = fs::status(file).permissions();
    
    // Check specific permissions
    bool owner_read = (p & fs::perms::owner_read) != fs::perms::none;
    bool owner_write = (p & fs::perms::owner_write) != fs::perms::none;
    bool owner_exec = (p & fs::perms::owner_exec) != fs::perms::none;
    
    // Set permissions
    fs::permissions(file, 
                    fs::perms::owner_all | fs::perms::group_read,
                    fs::perm_options::replace);
    
    // Add permissions
    fs::permissions(file,
                    fs::perms::owner_write,
                    fs::perm_options::add);
    
    // Remove permissions
    fs::permissions(file,
                    fs::perms::group_write,
                    fs::perm_options::remove);
    
    return 0;
}
```

### File Time Operations
```cpp
#include <filesystem>
#include <chrono>

namespace fs = std::filesystem;

int main() {
    fs::path file = "example.txt";
    
    // Get last write time
    auto ftime = fs::last_write_time(file);
    
    // Convert to system time
    auto sctp = std::chrono::time_point_cast<std::chrono::system_clock::duration>(
        ftime - fs::file_time_type::clock::now() + 
        std::chrono::system_clock::now()
    );
    std::time_t cftime = std::chrono::system_clock::to_time_t(sctp);
    std::cout << "Last modified: " << std::ctime(&cftime);
    
    // Set last write time
    fs::last_write_time(file, fs::file_time_type::clock::now());
    
    return 0;
}
```

---

## std::format (C++20)

Type-safe, Python-style formatting — the modern replacement for `sprintf` and most `stringstream` use cases. Check `__cpp_lib_format` before relying on it; libc++ and libstdc++ both ship it in recent releases.

### Basic Formatting
```cpp
#include <format>
#include <string>
#include <iostream>

int main() {
    // Basic formatting
    std::string s1 = std::format("Hello, {}!", "World");
    std::string s2 = std::format("Number: {}", 42);
    std::string s3 = std::format("Pi: {}", 3.14159);
    
    // Multiple arguments
    std::string s4 = std::format("{} + {} = {}", 2, 3, 2+3);
    
    // Positional arguments
    std::string s5 = std::format("{1} {0}", "World", "Hello");  // "Hello World"
    
    std::cout << s1 << std::endl;
    
    return 0;
}
```

### Format Specifications
```cpp
#include <format>
#include <iostream>

int main() {
    // Width and alignment
    std::cout << std::format("{:10}", "left") << std::endl;      // "left      "
    std::cout << std::format("{:<10}", "left") << std::endl;     // "left      "
    std::cout << std::format("{:>10}", "right") << std::endl;    // "     right"
    std::cout << std::format("{:^10}", "center") << std::endl;   // "  center  "
    
    // Fill character
    std::cout << std::format("{:*<10}", "left") << std::endl;    // "left******"
    std::cout << std::format("{:0>5}", 42) << std::endl;         // "00042"
    
    // Sign
    std::cout << std::format("{:+}", 42) << std::endl;           // "+42"
    std::cout << std::format("{: }", 42) << std::endl;           // " 42"
    
    // Number base
    std::cout << std::format("{:x}", 255) << std::endl;          // "ff"
    std::cout << std::format("{:X}", 255) << std::endl;          // "FF"
    std::cout << std::format("{:#x}", 255) << std::endl;         // "0xff"
    std::cout << std::format("{:o}", 64) << std::endl;           // "100"
    std::cout << std::format("{:b}", 5) << std::endl;            // "101"
    
    // Floating point
    std::cout << std::format("{:.2f}", 3.14159) << std::endl;    // "3.14"
    std::cout << std::format("{:.2e}", 3.14159) << std::endl;    // "3.14e+00"
    std::cout << std::format("{:.2g}", 3.14159) << std::endl;    // "3.1"
    
    // Precision and width
    std::cout << std::format("{:10.2f}", 3.14159) << std::endl;  // "      3.14"
    
    return 0;
}
```

### Custom Type Formatting
```cpp
#include <format>
#include <iostream>

struct Point {
    int x, y;
};

// Specialize std::formatter for Point
template<>
struct std::formatter<Point> {
    constexpr auto parse(std::format_parse_context& ctx) {
        return ctx.begin();
    }
    
    auto format(const Point& p, std::format_context& ctx) const {
        return std::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};

int main() {
    Point p{10, 20};
    std::cout << std::format("Point: {}", p) << std::endl;
    // Output: "Point: (10, 20)"
    
    return 0;
}
```

### std::print (C++23)

Direct formatted output without building an intermediate `std::string`. `std::println` appends a newline.

```cpp
#include <print>

int main() {
    // Direct printing (no intermediate string)
    std::print("Hello, {}!\n", "World");
    std::print("Number: {}\n", 42);
    
    // Print to stderr
    std::print(std::cerr, "Error: {}\n", "Something went wrong");
    
    // std::println adds '\n' automatically
    std::println("Done");
    
    return 0;
}
```

💡 **Hunch**: `std::print` requires `__cpp_lib_print` (C++23). Apple Clang/libc++ added it in Xcode 16+; older toolchains should use `std::cout << std::format(...)` instead.

---

## Formatting Comparison

```
┌───────────────────────────────────────────────────────┐
│         Formatting Methods Comparison                 │
├───────────────────────────────────────────────────────┤
│ Method          │ Type Safety │ Performance │ C++ Ver │
├─────────────────┼─────────────┼─────────────┼─────────┤
│ printf          │ No          │ Fast        │ C       │
│ std::cout <<    │ Yes         │ Slow        │ C++98   │
│ std::format     │ Yes         │ Fast        │ C++20   │
│ std::print      │ Yes         │ Fast        │ C++23   │
│                                                       │
│ Recommendation: Use std::format/print in modern C++   │
└───────────────────────────────────────────────────────┘
```

---

## Performance Tips

### Buffering
```cpp
#include <iostream>

int main() {
    // Unbuffered (slow)
    for (int i = 0; i < 10000; ++i) {
        std::cout << i << std::endl;  // Flushes every iteration
    }
    
    // Buffered (fast)
    for (int i = 0; i < 10000; ++i) {
        std::cout << i << '\n';  // No flush
    }
    std::cout << std::flush;  // Single flush at end
    
    // Disable sync with C stdio (faster)
    std::ios::sync_with_stdio(false);
    std::cin.tie(nullptr);
    
    return 0;
}
```

### Memory-Mapped Files (Platform-Specific)
```cpp
// For very large files, consider memory-mapped I/O
// Platform-specific (POSIX mmap, Windows CreateFileMapping)
```

---

## Best Practices

### 1. Always Check Stream State
```cpp
std::ifstream file("data.txt");
if (!file.is_open()) {
    std::cerr << "Failed to open file\n";
    return 1;
}
```

### 2. Use RAII for Files
```cpp
// GOOD: Automatic cleanup — see [Advanced Features](12_advanced_features.md)
{
    std::ofstream file("data.txt");
    file << "data";
}  // File closed automatically

// BAD: Manual management
FILE* f = fopen("data.txt", "w");
fprintf(f, "data");
fclose(f);  // Easy to forget
```

### 3. Prefer std::filesystem
```cpp
// GOOD: Portable, safe
fs::path p = fs::current_path() / "file.txt";
if (fs::exists(p)) { /* ... */ }

// BAD: Platform-specific, unsafe
#ifdef _WIN32
    // Windows code
#else
    // POSIX code
#endif
```

### 4. Use std::format for Complex Strings
```cpp
// GOOD: Clear, type-safe
auto msg = std::format("User {} logged in at {}", name, time);

// BAD: Error-prone
char buffer[256];
sprintf(buffer, "User %s logged in at %d", name.c_str(), time);
```

---

## Common Patterns

### Reading Entire File
```cpp
#include <fstream>
#include <string>
#include <sstream>

// Read as string
std::string read_file(const std::string& filename) {
    std::ifstream file(filename);
    std::stringstream buffer;
    buffer << file.rdbuf();
    return buffer.str();
}

// Read lines
std::vector<std::string> read_lines(const std::string& filename) {
    std::ifstream file(filename);
    std::vector<std::string> lines;
    std::string line;
    while (std::getline(file, line)) {
        lines.push_back(line);
    }
    return lines;
}
```

### Safe File Writing
```cpp
void write_file_safe(const std::string& filename, const std::string& data) {
    std::string temp = filename + ".tmp";
    
    // Write to temporary file
    {
        std::ofstream out(temp);
        if (!out) throw std::runtime_error("Cannot write to temp file");
        out << data;
        if (!out) throw std::runtime_error("Write failed");
    }
    
    // Rename (atomic on most systems)
    fs::rename(temp, filename);
}
```

### Directory Tree Walking
```cpp
void process_directory_tree(const fs::path& dir) {
    for (const auto& entry : fs::recursive_directory_iterator(dir)) {
        try {
            if (entry.is_regular_file()) {
                std::cout << "File: " << entry.path() 
                         << " (" << entry.file_size() << " bytes)\n";
            } else if (entry.is_directory()) {
                std::cout << "Dir:  " << entry.path() << "\n";
            }
        } catch (const fs::filesystem_error& e) {
            std::cerr << "Error: " << e.what() << "\n";
        }
    }
}
```

---

## Complete Practical Example: Log File Analyzer

Here's a comprehensive example integrating file I/O, filesystem operations, and formatting:

```cpp
#include <iostream>
#include <fstream>
#include <sstream>
#include <filesystem>
#include <string>
#include <vector>
#include <map>
#include <regex>
#include <chrono>
#include <iomanip>
#include <format>  // C++20

namespace fs = std::filesystem;

// Log entry structure
struct LogEntry {
    std::string timestamp;
    std::string level;
    std::string message;
    std::string source_file;
    int line_number;
};

// Log analyzer class
class LogAnalyzer {
private:
    std::map<std::string, int> level_counts_;
    std::map<std::string, int> file_counts_;
    std::vector<LogEntry> error_entries_;
    size_t total_lines_ = 0;
    
public:
    // Parse log line
    bool parse_line(const std::string& line, LogEntry& entry) {
        // Format: [2024-12-04 10:30:45] [ERROR] [file.cpp:123] Message here
        std::regex pattern(R"(\[([^\]]+)\]\s*\[(\w+)\]\s*\[([^:]+):(\d+)\]\s*(.+))");
        std::smatch matches;
        
        if (std::regex_match(line, matches, pattern)) {
            entry.timestamp = matches[1];
            entry.level = matches[2];
            entry.source_file = matches[3];
            entry.line_number = std::stoi(matches[4]);
            entry.message = matches[5];
            return true;
        }
        
        return false;
    }
    
    // Analyze a log file
    void analyze_file(const fs::path& file_path) {
        std::ifstream file(file_path);
        
        if (!file.is_open()) {
            std::cerr << "Failed to open: " << file_path << "\n";
            return;
        }
        
        std::string line;
        while (std::getline(file, line)) {
            total_lines_++;
            
            LogEntry entry;
            if (parse_line(line, entry)) {
                // Count by level
                level_counts_[entry.level]++;
                
                // Count by source file
                file_counts_[entry.source_file]++;
                
                // Store errors for detailed report
                if (entry.level == "ERROR" || entry.level == "CRITICAL") {
                    error_entries_.push_back(entry);
                }
            }
        }
    }
    
    // Analyze all log files in directory
    void analyze_directory(const fs::path& dir_path) {
        if (!fs::exists(dir_path) || !fs::is_directory(dir_path)) {
            std::cerr << "Invalid directory: " << dir_path << "\n";
            return;
        }
        
        std::cout << "Analyzing logs in: " << dir_path << "\n\n";
        
        // Iterate through .log files
        for (const auto& entry : fs::directory_iterator(dir_path)) {
            if (entry.is_regular_file() && entry.path().extension() == ".log") {
                std::cout << "Processing: " << entry.path().filename() << "\n";
                analyze_file(entry.path());
            }
        }
    }
    
    // Generate formatted report
    void generate_report(const std::string& output_file) {
        std::ofstream out(output_file);
        
        if (!out.is_open()) {
            std::cerr << "Failed to create report file\n";
            return;
        }
        
        // Write header with formatting
        out << std::string(60, '=') << "\n";
        out << std::setw(30) << "LOG ANALYSIS REPORT" << "\n";
        out << std::string(60, '=') << "\n\n";
        
        // Current timestamp
        auto now = std::chrono::system_clock::now();
        auto time = std::chrono::system_clock::to_time_t(now);
        out << "Generated: " << std::put_time(std::localtime(&time), 
                                             "%Y-%m-%d %H:%M:%S") << "\n\n";
        
        // Total lines
        out << "Total log lines processed: " << total_lines_ << "\n\n";
        
        // Log level distribution
        out << "=== Log Level Distribution ===\n";
        out << std::left;
        for (const auto& [level, count] : level_counts_) {
            double percentage = (100.0 * count) / total_lines_;
            out << std::setw(15) << level 
                << std::setw(8) << count 
                << std::fixed << std::setprecision(2)
                << "(" << percentage << "%)\n";
        }
        out << "\n";
        
        // Top source files
        out << "=== Top Source Files ===\n";
        
        // Convert to vector and sort
        std::vector<std::pair<std::string, int>> sorted_files(
            file_counts_.begin(), file_counts_.end()
        );
        std::sort(sorted_files.begin(), sorted_files.end(),
            [](const auto& a, const auto& b) {
                return a.second > b.second;
            });
        
        int top_n = std::min(10, (int)sorted_files.size());
        for (int i = 0; i < top_n; ++i) {
            out << std::setw(30) << sorted_files[i].first 
                << std::setw(8) << sorted_files[i].second << " entries\n";
        }
        out << "\n";
        
        // Error details
        if (!error_entries_.empty()) {
            out << "=== Error Details ===\n";
            for (const auto& entry : error_entries_) {
                out << "[" << entry.timestamp << "] "
                    << entry.level << " in "
                    << entry.source_file << ":" << entry.line_number << "\n"
                    << "  " << entry.message << "\n\n";
            }
        }
        
        out << std::string(60, '=') << "\n";
        
        std::cout << "\nReport generated: " << output_file << "\n";
    }
    
    // Export to CSV
    void export_to_csv(const std::string& csv_file) {
        std::ofstream out(csv_file);
        
        if (!out.is_open()) {
            std::cerr << "Failed to create CSV file\n";
            return;
        }
        
        // CSV header
        out << "Timestamp,Level,Source File,Line,Message\n";
        
        // CSV data (escape commas in messages)
        for (const auto& entry : error_entries_) {
            // Escape message if it contains comma or quote
            std::string escaped_msg = entry.message;
            if (escaped_msg.find(',') != std::string::npos ||
                escaped_msg.find('"') != std::string::npos) {
                escaped_msg = "\"" + escaped_msg + "\"";
            }
            
            out << entry.timestamp << ","
                << entry.level << ","
                << entry.source_file << ","
                << entry.line_number << ","
                << escaped_msg << "\n";
        }
        
        std::cout << "CSV exported: " << csv_file << "\n";
    }
    
    // Print statistics to console with formatting
    void print_statistics() {
        std::cout << "\n" << std::string(50, '=') << "\n";
        std::cout << "STATISTICS SUMMARY\n";
        std::cout << std::string(50, '=') << "\n";
        
        std::cout << std::format("Total lines: {:L}\n", total_lines_);
        
        std::cout << "\nBy Level:\n";
        for (const auto& [level, count] : level_counts_) {
            double pct = (100.0 * count) / total_lines_;
            std::cout << std::format("  {:<10} {:>6} ({:>5.1f}%)\n", 
                                    level, count, pct);
        }
        
        std::cout << std::format("\nErrors/Critical: {}\n", error_entries_.size());
        std::cout << std::string(50, '=') << "\n";
    }
};

// Create sample log files for demonstration
void create_sample_logs(const fs::path& dir) {
    // Create directory if it doesn't exist
    fs::create_directories(dir);
    
    // Sample log 1
    {
        std::ofstream log1(dir / "app.log");
        log1 << "[2024-12-04 10:00:00] [INFO] [main.cpp:10] Application started\n";
        log1 << "[2024-12-04 10:00:05] [DEBUG] [init.cpp:25] Initializing modules\n";
        log1 << "[2024-12-04 10:00:10] [WARNING] [config.cpp:45] Config file not found, using defaults\n";
        log1 << "[2024-12-04 10:00:15] [ERROR] [database.cpp:88] Connection failed: timeout\n";
        log1 << "[2024-12-04 10:00:20] [INFO] [main.cpp:50] Retrying connection\n";
    }
    
    // Sample log 2
    {
        std::ofstream log2(dir / "errors.log");
        log2 << "[2024-12-04 10:01:00] [ERROR] [network.cpp:120] Socket error: connection refused\n";
        log2 << "[2024-12-04 10:02:00] [CRITICAL] [memory.cpp:200] Out of memory\n";
        log2 << "[2024-12-04 10:03:00] [ERROR] [parser.cpp:75] Invalid JSON format\n";
    }
    
    std::cout << "Sample log files created in: " << dir << "\n\n";
}

// Backup log files
void backup_logs(const fs::path& src_dir, const fs::path& backup_dir) {
    fs::create_directories(backup_dir);
    
    for (const auto& entry : fs::directory_iterator(src_dir)) {
        if (entry.is_regular_file() && entry.path().extension() == ".log") {
            // Create timestamped backup
            auto now = std::chrono::system_clock::now();
            auto time = std::chrono::system_clock::to_time_t(now);
            
            std::ostringstream backup_name;
            backup_name << entry.path().stem().string() << "_"
                       << std::put_time(std::localtime(&time), "%Y%m%d_%H%M%S")
                       << entry.path().extension().string();
            
            fs::path backup_path = backup_dir / backup_name.str();
            fs::copy_file(entry.path(), backup_path, 
                         fs::copy_options::overwrite_existing);
            
            std::cout << "Backed up: " << entry.path().filename() 
                      << " -> " << backup_path.filename() << "\n";
        }
    }
}

int main() {
    std::cout << "=== Log File Analyzer ===\n\n";
    
    // Setup directories
    fs::path log_dir = "sample_logs";
    fs::path backup_dir = "backup_logs";
    fs::path report_dir = "reports";
    
    // Create sample logs
    create_sample_logs(log_dir);
    
    // Analyze logs
    LogAnalyzer analyzer;
    analyzer.analyze_directory(log_dir);
    
    // Print statistics
    analyzer.print_statistics();
    
    // Generate reports
    fs::create_directories(report_dir);
    analyzer.generate_report(report_dir / "analysis_report.txt");
    analyzer.export_to_csv(report_dir / "errors.csv");
    
    // Backup logs
    backup_logs(log_dir, backup_dir);
    
    // Show directory sizes
    std::cout << "\n=== Directory Sizes ===\n";
    for (const auto& dir : {log_dir, backup_dir, report_dir}) {
        if (fs::exists(dir)) {
            uintmax_t total_size = 0;
            for (const auto& entry : fs::recursive_directory_iterator(dir)) {
                if (entry.is_regular_file()) {
                    total_size += entry.file_size();
                }
            }
            std::cout << std::format("{:<15} {:>8} bytes\n", 
                                    dir.string(), total_size);
        }
    }
    
    std::cout << "\n=== Analysis Complete ===\n";
    
    return 0;
}
```

### Concepts Demonstrated:
- **File I/O**:
  - `std::ifstream` for reading
  - `std::ofstream` for writing
  - `std::getline()` for line-by-line reading
  - Binary vs text mode
- **std::filesystem**:
  - `directory_iterator` for listing files
  - `create_directories()` for directory creation
  - `copy_file()` for backups
  - `file_size()` and `is_regular_file()`
  - Path manipulation
- **Formatting**:
  - `std::format` (C++20) for modern formatting
  - `std::setw`, `std::setprecision` for alignment
  - `std::put_time` for timestamp formatting
  - CSV generation with escaping
- **String streams**:
  - `std::ostringstream` for building strings
  - `std::istringstream` for parsing
- **Regex**: Pattern matching for log parsing
- **Stream manipulators**: Formatting output

This example shows real-world file and filesystem operations!

---

## Related Topics

- [Exception Handling](17_exceptions.md) — `std::filesystem::filesystem_error`, `std::error_code` overloads, and I/O exception safety
- [Regex](20_regex.md) — pattern matching for log parsing and text processing (used in the example)
- [Time and Chrono](18_time_chrono.md) — timestamp formatting with `std::chrono` and `std::put_time`
- [Async and Futures](15_async_futures.md) — parallel file processing with `std::async`
- [Advanced Features](12_advanced_features.md) — RAII for streams and move semantics for `fstream`
- [Modern C++20/23 Features](07_modern_features.md) — `std::format`, `std::print`, and other formatting additions
- [Quick Reference](99_quick_reference.md) — I/O and filesystem APIs at a glance

## Next Steps
- **Next**: [Exception Handling →](17_exceptions.md)
- **Previous**: [← Async Programming](15_async_futures.md)

---
*Chapter 16 — I/O, Filesystem, and Formatting*

