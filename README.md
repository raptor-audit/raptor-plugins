# Raptor Plugins

Official plugin repository for [Raptor](https://github.com/calvin-kimani/raptor) - Smart Contract Audit Framework.

## Creating a Plugin

### Plugin Structure

Every plugin must have an `install.py` file as the entry point:

```
my-plugin/
├── install.py          # Required: Plugin metadata and installation
├── __init__.py         # Optional: Package initialization
├── __main__.py         # Optional: CLI entry point
├── README.md           # Recommended: Plugin documentation
└── [other files]       # Plugin implementation
```

### Minimal install.py

```python
"""
My Plugin - Brief description
"""

from pathlib import Path

# Required: Plugin metadata
TOOL_INFO = {
    "name": "my-plugin",
    "description": "Brief description of what the plugin does",
    "version": "1.0.0",
    "type": "library",  # or "cli" or "library-cli"

    # Optional fields
    "provides": ["MyClass", "helper_function"],  # Exported symbols
    "depends_on": [                              # Plugin dependencies
        "solidity-parser>=1.0.0",                # Version constraint
        "other-plugin"                           # Any version
    ],
    "cli_command": "python -m my_plugin",
    "cli_usage": "python -m my_plugin <args>",
}

# Required: Installation function
def install(install_dir: Path = None) -> bool:
    """
    Install the plugin.

    Args:
        install_dir: Directory where plugin is installed

    Returns:
        True if installation succeeded
    """
    if install_dir is None:
        install_dir = Path(__file__).parent

    print(f"[my-plugin] Installing to {install_dir}")

    try:
        # Install dependencies, setup, etc.
        print(f"[my-plugin] Installation complete")
        return True
    except Exception as e:
        print(f"[my-plugin] Installation failed: {e}")
        return False

# Optional: Check installation status
def check() -> dict:
    """
    Check if plugin components are installed.

    Returns:
        Dict mapping component names to installation status (bool)
    """
    return {
        "core": True,
        "optional_feature": False,
    }

# Optional: Register CLI commands
def register_commands(subparsers):
    """
    Register CLI commands with Raptor.

    Args:
        subparsers: argparse subparsers object

    Returns:
        Dict mapping command names to handler functions
    """
    # Add 'raptor mycommand' subcommand
    mycommand_parser = subparsers.add_parser(
        'mycommand',
        help='Short help text',
        description='Detailed description'
    )
    mycommand_parser.add_argument('input', help='Input file')

    def mycommand_handler(args):
        """Handle 'raptor mycommand' execution."""
        print(f"Processing {args.input}")
        return 0  # Return exit code

    return {'mycommand': mycommand_handler}

# Optional: Uninstall hook
def uninstall(install_dir: Path) -> bool:
    """Cleanup when plugin is uninstalled."""
    print(f"[my-plugin] Uninstalling from {install_dir}")
    return True
```

### Plugin Types

**Library Plugin**: Provides reusable functionality
```python
TOOL_INFO = {
    "name": "solidity-parser",
    "type": "library",
    "provides": ["SolidityParser", "ContractInfo"],
}
```

**CLI Plugin**: Adds new Raptor commands
```python
TOOL_INFO = {
    "name": "my-tool",
    "type": "cli",
}

def register_commands(subparsers):
    # Register 'raptor mytool' command
    pass
```

**Library + CLI**: Both library and CLI
```python
TOOL_INFO = {
    "name": "solidity-parser",
    "type": "library-cli",
    "provides": ["SolidityParser"],
}
```

### Dependency Management

Declare plugin dependencies with version constraints:

```python
TOOL_INFO = {
    "name": "boundary-analyzer",
    "depends_on": [
        "solidity-parser>=1.0.0",      # Minimum version
        "graph-builder==2.0.0",        # Exact version
        "utility-plugin",              # Any version
    ],
}
```

**Version Operators**: `>=`, `>`, `<=`, `<`, `==`

Dependencies are auto-installed when your plugin is installed. Access them by loading from their directory:

```python
import sys
from pathlib import Path

# Add dependency to path
plugin_base = Path(__file__).parent.parent
sys.path.insert(0, str(plugin_base / 'solidity-parser'))
from parser import SolidityParser
```

### Specifying Dependency URLs

You can specify custom URLs for dependencies:

```python
TOOL_INFO = {
    "depends_on": [
        "custom-plugin@https://raw.githubusercontent.com/user/repo/main/plugin/install.py",
        "local-plugin@file:///path/to/local/plugin",
        "relative-plugin@./plugins/my-plugin",
    ],
}
```

### Installing Python Packages

Install required packages in your `install()` function:

```python
import subprocess
import sys

def install(install_dir: Path = None) -> bool:
    try:
        # Install required packages
        packages = ["numpy", "requests"]
        for package in packages:
            subprocess.run(
                [sys.executable, '-m', 'pip', 'install', package],
                check=True,
                capture_output=True
            )
            print(f"[my-plugin] ✓ Installed {package}")

        return True
    except subprocess.CalledProcessError:
        print(f"[my-plugin] ✗ Failed to install {package}")
        return False
```

### Registering CLI Commands

Register commands with detailed arguments:

```python
def register_commands(subparsers):
    """Register 'raptor mycommand' command."""

    parser = subparsers.add_parser(
        'mycommand',
        help='Short help text',
        description='Detailed description'
    )

    # Add arguments
    parser.add_argument('input', help='Input file or directory')
    parser.add_argument('--format', choices=['json', 'yaml'], default='json')
    parser.add_argument('--output', '-o', help='Output file')
    parser.add_argument('--quiet', action='store_true')

    def mycommand_handler(args):
        """Handle command execution."""
        print(f"Processing {args.input}")
        # Do work here
        return 0  # Exit code

    return {'mycommand': mycommand_handler}
```

For complex commands, delegate to `__main__.py`:

```python
def register_commands(subparsers):
    parser = subparsers.add_parser('parse', help='Parse Solidity files')
    parser.add_argument('paths', nargs='+')

    def parse_handler(args):
        """Delegate to __main__.py."""
        import importlib.util
        plugin_dir = Path(__file__).parent
        main_file = plugin_dir / '__main__.py'

        spec = importlib.util.spec_from_file_location("plugin_main", main_file)
        main_module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(main_module)

        saved_argv = sys.argv
        sys.argv = ['plugin'] + args.paths
        result = main_module.main()
        sys.argv = saved_argv
        return result

    return {'parse': parse_handler}
```

### Multi-Version Support

Raptor supports multiple versions of the same plugin installed side-by-side:

```
.plugins/
└── my-plugin/
    ├── 1.0.0/          # Version 1.0.0
    ├── 1.1.0/          # Version 1.1.0
    └── 2.0.0/          # Version 2.0.0 (active)
```

Commands:
```bash
# Install plugin (becomes active)
raptor plugins install my-plugin

# Install specific version
raptor plugins install my-plugin  # Downloads latest

# Switch active version
raptor plugins switch my-plugin 2.0.0

# List versions
raptor plugins list
```

### Testing Your Plugin

Test locally before publishing:

```bash
# 1. Add plugin to raptor.toml
cat >> raptor.toml << EOF
[plugins.my-plugin]
url = "file:///path/to/my-plugin/install.py"
description = "My test plugin"
EOF

# 2. Install plugin
raptor plugins install my-plugin

# 3. Test functionality
raptor plugins list
raptor mycommand --help  # If plugin registers commands

# 4. Reinstall after changes
raptor plugins install my-plugin --force
```

### Publishing Your Plugin

1. **Host install.py** on a public URL:
   - GitHub: `https://raw.githubusercontent.com/user/repo/main/plugins/my-plugin/install.py`
   - GitLab: `https://gitlab.com/user/repo/-/raw/main/plugins/my-plugin/install.py`

2. **Add to registry** (raptor.toml):
```toml
[plugins.my-plugin]
url = "https://raw.githubusercontent.com/user/repo/main/plugins/my-plugin/install.py"
version = ">=1.0.0"  # Optional constraint
description = "Brief description"
```

3. **Version with semantic versioning** (MAJOR.MINOR.PATCH):
```python
TOOL_INFO = {
    "version": "1.2.3",  # Update for each release
}
```

### Best Practices

1. **Error Handling**: Always handle errors gracefully
```python
def install(install_dir: Path = None) -> bool:
    try:
        # Installation steps
        return True
    except Exception as e:
        print(f"[my-plugin] Installation failed: {e}")
        import traceback
        traceback.print_exc()
        return False
```

2. **User Feedback**: Provide clear status messages
```python
print(f"[my-plugin] Installing dependencies...")
print(f"[my-plugin] ✓ Installed numpy")
print(f"[my-plugin] ✗ Failed to install graphviz")
print(f"[my-plugin] ⚠ Warning: solc not found")
print(f"[my-plugin] ℹ Optional feature disabled")
```

3. **Optional Dependencies**: Handle missing optionals gracefully
```python
try:
    import graphviz
    GRAPHVIZ_AVAILABLE = True
except ImportError:
    GRAPHVIZ_AVAILABLE = False
    print("[my-plugin] ⚠ graphviz not available, visualization disabled")
```

4. **Documentation**: Add docstrings and README
5. **Testing**: Test with different Python versions and inputs
6. **Compatibility**: Use semantic versioning for breaking changes

## License

This repository is private and proprietary.
