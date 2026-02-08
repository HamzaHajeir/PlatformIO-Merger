# PlatformIO INI Configuration Merger

A Python tool for intelligently merging PlatformIO INI configuration files with three-phase overlay support. Handles multi-line values, variable references, and PlatformIO-specific formatting.

## Features

- **Three-Phase Overlay System**: REMOVE, SUBSTITUTE, and INSERT operations
- **Intelligent Line Merging**: Appends missing lines for multi-value keys instead of replacing
- **Variable Reference Handling**: Automatically cleans up dangling `${section.key}` references
- **PlatformIO-Aware**: Preserves proper indentation (tabs) and formatting
- **Recursive Cleanup**: Removes empty items and sections after operations
- **Comprehensive Validation**: File existence, permissions, and syntax checking

## Installation

No installation required - just download and run:

```bash
git clone https://github.com/HamzaHajeir/PlatformIO-Merger
cd platformio-merger
```

Requirements: Python 3.6+

## Usage

### Command Line Interface

```bash
# Basic merge
python platformio_merger.py --base platformio.ini --overlay changes.ini --output merged.ini

# With verbose output
python platformio_merger.py --base platformio.ini --overlay changes.ini --output merged.ini --verbose

# Prevent overwriting existing output
python platformio_merger.py --base platformio.ini --overlay changes.ini --output merged.ini --no-clobber

# Generate example files
python platformio_merger.py --example
python platformio_merger.py --example ./my_example_dir

# Show help
python platformio_merger.py --help

# Show version
python platformio_merger.py --version
```

### Overlay File Format

Overlay files contain three optional partitions marked with comment headers:

```ini
; === REMOVE ===
; Remove specific lines or entire keys
[common]
lib_deps = oldlib          ; Remove lines containing "oldlib"
monitor_speed =            ; Remove entire monitor_speed key

[env:esp32]
board_build.partitions =   ; Remove entire partitions key

; === SUBSTITUTE ===
; Replace entire values
[common]
monitor_speed = 115200     ; Replace monitor_speed value

[env:esp32]
board = esp32devkitv1      ; Replace board value

; === INSERT ===
; Append lines to existing keys or create new keys
[common]
lib_deps =                 ; Append to existing lib_deps
    https://github.com/owner/newlib#1.0.0
build_flags =              ; Append to existing build_flags
    -DOPTIMIZED

[env:pico]                 ; Create new environment section
board = pico_w
platform = raspberrypi
```

**Note**: If no markers are found, the entire file is treated as an INSERT partition.

## Python API

### Import and Use

```python
from platformio_merger import merge_files

# Simple merge
success = merge_files(
    base_file="platformio.ini",
    overlay_file="changes.ini",
    output_file="merged.ini"
)

# With options
success = merge_files(
    base_file="platformio.ini",
    overlay_file="changes.ini",
    output_file="merged.ini",
    verbose=True,      # Print progress to stderr
    no_clobber=False   # Allow overwriting output file
)
```

### Advanced Usage

```python
from platformio_merger import PlatformIOMerger

# Create merger instance
merger = PlatformIOMerger(verbose=True)

# Perform merge with detailed results
result = merger.merge("base.ini", "overlay.ini", "output.ini")

if result['success']:
    print("Merge statistics:", result['stats'])
    print("Warnings:", result['warnings'])
else:
    print("Errors:", result['errors'])
```

## Merge Behavior

### Processing Order
1. **REMOVE Phase**: Remove specified lines or entire keys
2. **SUBSTITUTE Phase**: Replace entire values
3. **INSERT Phase**: Append lines to existing keys or create new keys
4. **Cleanup Phase**: Remove empty items and dangling references

### Line Matching Rules
- **Exact matching**: For REMOVE operations, uses substring matching
- **Case-sensitive**: All operations are case-sensitive
- **Whitespace trimmed**: Leading/trailing whitespace ignored for matching
- **Comments stripped**: Inline comments (`;` or `#`) ignored for matching

### Multi-line Value Handling
For keys like `lib_deps`, `build_flags`, `platform_packages`:
- **INSERT**: Appends new lines that don't already exist
- **REMOVE**: Removes lines containing specified patterns
- **SUBSTITUTE**: Replaces entire multi-line value

### Variable Reference Handling
- `${section.key}` references are preserved
- If referenced item is removed, the reference is also removed
- Circular reference detection with safety limit (100 iterations)

## Examples

### Example 1: Adding a Library Dependency

**base.ini:**
```ini
[common]
lib_deps = 
    https://github.com/owner/lib1#1.0.0
```

**overlay.ini:**
```ini
; === INSERT ===
[common]
lib_deps = 
    https://github.com/owner/lib2#2.0.0
```

**merged.ini:**
```ini
[common]
lib_deps = 
    https://github.com/owner/lib1#1.0.0
    https://github.com/owner/lib2#2.0.0
```

### Example 2: Removing and Replacing

**base.ini:**
```ini
[common]
monitor_speed = 921600
build_flags = 
    -DDEBUG
    -DVERBOSE
```

**overlay.ini:**
```ini
; === REMOVE ===
[common]
build_flags = -DDEBUG

; === SUBSTITUTE ===
[common]
monitor_speed = 115200
```

**merged.ini:**
```ini
[common]
monitor_speed = 115200
build_flags = 
    -DVERBOSE
```

## Error Handling

### Exit Codes
- `0`: Success
- `1`: File/IO errors, parse errors, merge errors
- `2`: Usage errors (invalid arguments, missing files)
- `3`: Overwrite prevented by `--no-clobber`

### Common Errors
- **File not found**: Check file paths and permissions
- **Parse error**: Invalid INI syntax in input files
- **Circular reference**: Variable references that create infinite loops
- **Permission denied**: Output directory not writable

## Limitations

### Current Limitations
1. **No inheritance resolution**: Does not resolve `extends` directives in PlatformIO environments
2. **Single overlay file**: Only one overlay file supported (multiple overlays planned for v2)
3. **No dry-run mode**: Cannot preview changes without writing output file
4. **Basic pattern matching**: REMOVE uses simple substring matching only
5. **No backup creation**: Does not create backups of original files

### PlatformIO-Specific Notes
- Uses `=` as only delimiter (not `:`)
- Preserves case sensitivity for keys
- Formats multi-line values with tabs (not spaces)
- Does not validate PlatformIO-specific syntax

## Advanced Topics

### Custom Integration

```python
import subprocess

# Call as subprocess
result = subprocess.run([
    "python", "platformio_merger.py",
    "--base", "platformio.ini",
    "--overlay", "changes.ini", 
    "--output", "merged.ini",
    "--verbose"
], capture_output=True, text=True)

if result.returncode == 0:
    print("Success:", result.stdout)
else:
    print("Error:", result.stderr)
```

### Extending for Custom Use Cases

```python
class CustomPlatformIOMerger(PlatformIOMerger):
    """Extended merger with custom behavior."""
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # Add custom append-only keys
        self.custom_append_keys = {'custom_multi_value'}
    
    def _append_lines_to_value(self, base_value, lines_to_add):
        # Custom merging logic
        # ... implementation ...
        pass
```

## Development

### Running Tests
```bash
# Generate and test example files
python platformio_merger.py --example
python platformio_merger.py --base piomerger_example/base.ini --overlay piomerger_example/overlay.ini --output test.ini --verbose
```

### Project Structure
```
platformio_merger.py      # Main implementation
piomerger_example/        # Generated example directory
  base.ini               # Example base configuration
  overlay.ini            # Example overlay with all three phases
  merged.ini             # Expected output
```

## Roadmap (v2.0)

Planned features:
- Multiple overlay file support
- Inheritance resolution for `extends` directives
- Dry-run mode with diff output
- Regular expression support for REMOVE patterns
- Backup creation with rollback capability
- JSON output format for machine parsing
- Configuration file for default settings

## License

MIT LICENSE 2026 HamzaHajeir 

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Submit a pull request

## Support

For issues and feature requests, please use the GitHub issue tracker.
