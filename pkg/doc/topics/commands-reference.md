---
Title: Glazed Commands Reference
Slug: commands-reference
Short: Complete reference for creating, configuring, and running commands in Glazed
Topics:
- commands
- interfaces
- flags
- arguments
- layers
- dual-commands
IsTemplate: false
IsTopLevel: true
ShowPerDefault: true
SectionType: GeneralTopic
---

# Glazed Commands Reference

## Overview

The Glazed command system provides a structured approach to building CLI applications that handle multiple output formats, complex parameter validation, and reusable components. This reference covers the complete command system architecture, interfaces, and implementation patterns.

Building CLI tools typically involves handling parameter parsing, validation, output formatting, and configuration management. Glazed addresses these concerns through a layered architecture that separates command logic from presentation and parameter management.

The core principle is separation of concerns: commands focus on business logic while Glazed handles parameter parsing, validation, and output formatting. This approach enables automatic support for multiple output formats, consistent parameter handling across commands, and reusable parameter groups.

## Architecture of Glazed Commands

```
┌─────────────────────────────────────────────┐
│                Command Interface             │
├────────────┬─────────────────┬──────────────┤
│BareCommand │ WriterCommand   │ GlazeCommand │
└────────────┴─────────────────┴──────────────┘
                      │
            ┌─────────┼─────────┐
            ▼         ▼         ▼
┌─────────────────────────────────────────────┐
│          CommandDescription                  │
│  (name, flags, arguments, layers, etc.)     │
└─────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│               Parameter Layers               │
│     ┌───────────┬────────────┬────────┐     │
│     │Default    │Glazed      │Custom  │     │
│     │(flags/args)│(output fmt)│(any)   │     │
│     └───────────┴────────────┴────────┘     │
└─────────────────────────────────────────────┘
                │        │
      ┌─────────┘        └────────┐
      ▼                           ▼
┌──────────────┐          ┌─────────────────┐
│  Parameters  │          │ ParsedLayers    │
│ (definitions)│─────────▶│(runtime values) │
└──────────────┘          └─────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────┐
│     Execution (via Run methods or runner)    │
└─────────────────────────────────────────────┘
                 │
     ┌───────────┼────────────┐
     ▼           ▼            ▼
┌────────┐ ┌──────────┐ ┌──────────────┐
│Direct  │ │Writer    │ │GlazeProcessor│
│Output  │ │Output    │ │(structured)  │
└────────┘ └──────────┘ └──────────────┘
                 │            │
                 └─────┬──────┘
                       ▼
            ┌─────────────────────┐
            │   Dual Commands     │
            │ (can run in both    │
            │ classic & glaze)    │
            └─────────────────────┘
```

Key components:

1. **Command Interfaces**: Define how commands produce output - direct output (BareCommand), text streams (WriterCommand), or structured data (GlazeCommand)
2. **CommandDescription**: Contains metadata about a command (name, description, parameters, etc.)
3. **Parameter Layers**: Organize parameters into logical groups (database, logging, output, etc.)
4. **Parameters**: Define command inputs with type information, validation, and help text
5. **ParsedLayers**: Runtime values after collecting from CLI flags, environment, config files, and defaults
6. **Execution Methods**: Different approaches for running commands - CLI integration, programmatic execution, etc.

## Core Packages

Glazed's command functionality is distributed across several packages:

- `github.com/go-go-golems/glazed/pkg/cmds`: Core command interfaces and descriptions
- `github.com/go-go-golems/glazed/pkg/cmds/parameters`: Parameter types and definitions
- `github.com/go-go-golems/glazed/pkg/cmds/layers`: Parameter layering system
- `github.com/go-go-golems/glazed/pkg/cmds/runner`: Programmatic command execution
- `github.com/go-go-golems/glazed/pkg/middlewares`: Output processing
- `github.com/go-go-golems/glazed/pkg/types`: Structured data types
- `github.com/go-go-golems/glazed/pkg/cli`: Helpers for integrating with Cobra
- `github.com/go-go-golems/glazed/pkg/settings`: Standard Glazed parameter layers

## Command Interfaces

Glazed provides three main command interfaces, each designed for different output scenarios:

1. **BareCommand**: For commands that handle their own output directly
2. **WriterCommand**: For commands that write to a provided writer (like a file or console)  
3. **GlazeCommand**: For commands that produce structured data using Glazed's processing pipeline

Additionally, **Dual Commands** can implement multiple interfaces and switch between output modes at runtime based on flags.

### BareCommand

BareCommand provides direct control over output. Commands implementing this interface handle their own output formatting and presentation.

```go
// BareCommand interface definition
type BareCommand interface {
    Command
    Run(ctx context.Context, parsedLayers *layers.ParsedLayers) error
}
```

**Use cases:**
- Commands that print progress updates or status messages
- Utilities that perform actions (file operations, service management) rather than data processing
- Commands requiring custom output formatting
- Quick prototypes and simple utilities

**Example implementation:**
```go
type CleanupCommand struct {
    *cmds.CommandDescription
}

func (c *CleanupCommand) Run(ctx context.Context, parsedLayers *layers.ParsedLayers) error {
    s := &CleanupSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    fmt.Printf("Starting cleanup in %s...\n", s.Directory)
    
    files, err := findOldFiles(s.Directory, s.OlderThan)
    if err != nil {
        return fmt.Errorf("failed to scan directory: %w", err)
    }
    
    if len(files) == 0 {
        fmt.Printf("Directory is clean - no files older than %s found.\n", s.OlderThan)
        return nil
    }
    
    fmt.Printf("Found %d files to clean up:\n", len(files))
    for i, file := range files {
        fmt.Printf("  %d. %s\n", i+1, file)
        if !s.DryRun {
            if err := os.Remove(file); err != nil {
                fmt.Printf("     Failed to remove: %s\n", err)
            } else {
                fmt.Printf("     Removed\n")
            }
        }
    }
    
    if s.DryRun {
        fmt.Printf("Dry run completed. Use --execute to actually remove files.\n")
    } else {
        fmt.Printf("Cleanup completed successfully.\n")
    }
    
    return nil
}
```

### WriterCommand

WriterCommand allows commands to write text output to any destination (files, stdout, network connections) without knowing the specific target.

```go
// WriterCommand interface definition
type WriterCommand interface {
    Command
    RunIntoWriter(ctx context.Context, parsedLayers *layers.ParsedLayers, w io.Writer) error
}
```

This interface separates content generation from output destination, improving testability and reusability.

**Use cases:**
- Report generators that output to files or stdout
- Log processors that transform and forward data
- Commands generating substantial text output (documentation, configuration files)
- Commands where output destination should be configurable

**Example implementation:**
```go
type HealthReportCommand struct {
    *cmds.CommandDescription
}

func (c *HealthReportCommand) RunIntoWriter(ctx context.Context, parsedLayers *layers.ParsedLayers, w io.Writer) error {
    s := &HealthReportSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    // Generate a comprehensive system health report
    fmt.Fprintf(w, "System Health Report\n")
    fmt.Fprintf(w, "Generated: %s\n", time.Now().Format(time.RFC3339))
    fmt.Fprintf(w, "Host: %s\n\n", s.Hostname)
    
    // Check various system components
    components := []string{"CPU", "Memory", "Disk", "Network"}
    for _, component := range components {
        status, details := checkComponentHealth(component)
        fmt.Fprintf(w, "%s Status: %s\n", component, status)
        if s.Verbose {
            fmt.Fprintf(w, "  Details: %s\n", details)
        }
    }
    
    // Add recommendations if any issues found
    if recommendations := generateRecommendations(); len(recommendations) > 0 {
        fmt.Fprintf(w, "\nRecommendations:\n")
        for i, rec := range recommendations {
            fmt.Fprintf(w, "%d. %s\n", i+1, rec)
        }
    }
    
    return nil
}
```

This command can write its report to a file for archival, to stdout for immediate viewing, or even to a network connection for monitoring systems. The command logic doesn't change - only the destination.

### GlazeCommand

GlazeCommand produces structured data that Glazed processes into various output formats. Commands generate data rows instead of formatted text, enabling automatic format conversion and data processing.

```go
// GlazeCommand interface definition
type GlazeCommand interface {
    Command
    RunIntoGlazeProcessor(ctx context.Context, parsedLayers *layers.ParsedLayers, gp middlewares.Processor) error
}
```

GlazeCommand generates structured data events that can be transformed, filtered, sorted, and formatted automatically.

**Key capabilities:**
- **Automatic formatting**: JSON, YAML, CSV, HTML tables, custom templates without additional code
- **Data processing**: Built-in filtering, sorting, column selection, and transformation
- **Composability**: Output can be piped to other tools or processed programmatically
- **Format independence**: New output formats can be added without changing command implementation

**Real-world example - A server monitoring command:**
```go
type MonitorServersCommand struct {
    *cmds.CommandDescription
}

func (c *MonitorServersCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    s := &MonitorSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    // Get server data from various sources
    servers := getServersFromInventory(s.Environment)
    
    for _, server := range servers {
        // Check server health
        health := checkServerHealth(server.Hostname)
        
        // Produce a rich data row with nested information
        row := types.NewRow(
            types.MRP("hostname", server.Hostname),
            types.MRP("environment", server.Environment),
            types.MRP("cpu_percent", health.CPUPercent),
            types.MRP("memory_used_gb", health.MemoryUsedGB),
            types.MRP("memory_total_gb", health.MemoryTotalGB),
            types.MRP("disk_used_percent", health.DiskUsedPercent),
            types.MRP("status", health.Status),
            types.MRP("last_seen", health.LastSeen),
            types.MRP("alerts", health.ActiveAlerts), // Can be an array
            types.MRP("metadata", map[string]interface{}{ // Nested objects work too
                "os_version": server.OSVersion,
                "kernel": server.KernelVersion,
                "uptime_days": health.UptimeDays,
            }),
        )
        
        if err := gp.AddRow(ctx, row); err != nil {
            return err
        }
    }
    
    return nil
}
```

Now your users can run:
- `monitor --output table` for a human-readable overview
- `monitor --output json | jq '.[] | select(.status != "healthy")'` to find problem servers  
- `monitor --output csv > servers.csv` to import into spreadsheets
- `monitor --filter 'cpu_percent > 80' --sort cpu_percent` to find CPU hotspots
- `monitor --template custom.tmpl` to generate custom reports

All from the same command implementation.

### Dual Commands

Dual commands implement multiple interfaces and switch between output modes based on runtime flags. This approach provides both human-readable text output and structured data from a single command.

Dual commands address the need for different output formats: interactive use typically requires readable text output, while scripts need structured data. Rather than maintaining separate commands, dual commands adapt their behavior based on context.

```go
// Dual command example
type StatusCommand struct {
    *cmds.CommandDescription
}

// Implement BareCommand for classic mode
func (c *StatusCommand) Run(ctx context.Context, parsedLayers *layers.ParsedLayers) error {
    s := &StatusSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    // Human-readable output
    fmt.Printf("System Status:\n")
    fmt.Printf("  CPU: %.1f%%\n", s.CPUUsage)
    fmt.Printf("  Memory: %s\n", s.MemoryUsage)
    return nil
}

// Implement GlazeCommand for structured output mode
func (c *StatusCommand) RunIntoGlazeProcessor(
    ctx context.Context, 
    parsedLayers *layers.ParsedLayers, 
    gp middlewares.Processor,
) error {
    s := &StatusSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    // Structured data output
    row := types.NewRow(
        types.MRP("cpu_usage", s.CPUUsage),
        types.MRP("memory_usage", s.MemoryUsage),
        types.MRP("timestamp", time.Now()),
    )
    return gp.AddRow(ctx, row)
}

// Ensure both interfaces are implemented
var _ cmds.BareCommand = &StatusCommand{}
var _ cmds.GlazeCommand = &StatusCommand{}
```

## Command Implementation

Glazed commands follow a consistent structure with four key components:

### Command Structure

**Command Struct**: Contains the command's identity and embeds `CommandDescription` which holds metadata (name, flags, help text) separately from business logic.

**Settings Struct**: Provides type safety by defining a struct that mirrors command inputs. Glazed automatically maps parameters to struct fields.

**Run Method**: Contains business logic. The method signature depends on the implemented interface, but the pattern is consistent: extract settings, execute logic, return results.

**Constructor Function**: Creates the command description with its parameters and layers.

Implementation pattern:

```go
// Step 1: Define command struct
type MyCommand struct {
    *cmds.CommandDescription
}

// Step 2: Define settings struct
type MyCommandSettings struct {
    Count  int    `glazed.parameter:"count"`
    Format string `glazed.parameter:"format"`
    Verbose bool  `glazed.parameter:"verbose"`
}

// Step 3: Implement run method
func (c *MyCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    s := &MyCommandSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return err
    }
    
    // Command logic here
    for i := 0; i < s.Count; i++ {
        row := types.NewRow(types.MRP("id", i))
        if err := gp.AddRow(ctx, row); err != nil {
            return err
        }
    }
    
    return nil
}

// Step 4: Constructor function
func NewMyCommand() (*MyCommand, error) {
    glazedLayer, err := settings.NewGlazedParameterLayers()
    if err != nil {
        return nil, err
    }
    
    cmdDesc := cmds.NewCommandDescription(
        "mycommand",
        cmds.WithShort("My example command"),
        cmds.WithFlags(
            parameters.NewParameterDefinition(
                "count",
                parameters.ParameterTypeInteger,
                parameters.WithDefault(10),
                parameters.WithHelp("Number of items to generate"),
            ),
            parameters.NewParameterDefinition(
                "format",
                parameters.ParameterTypeChoice,
                parameters.WithChoices("json", "yaml", "table"),
                parameters.WithDefault("table"),
                parameters.WithHelp("Output format"),
            ),
            parameters.NewParameterDefinition(
                "verbose",
                parameters.ParameterTypeBool,
                parameters.WithDefault(false),
                parameters.WithHelp("Enable verbose output"),
            ),
        ),
        cmds.WithLayersList(glazedLayer),
    )
    
    return &MyCommand{
        CommandDescription: cmdDesc,
    }, nil
}

// Ensure interface compliance
var _ cmds.GlazeCommand = &MyCommand{}
```

## Parameters

Glazed parameters are typed objects with validation rules and behavior, unlike traditional CLI libraries that treat parameters as simple strings requiring manual parsing and validation. This enables automatic validation, help generation, and multi-source value loading.

### Parameter Type System

Parameter types define data structure, parsing behavior, and validation rules. Each type handles string parsing, validation, and help text generation.

#### Basic Types
**`ParameterTypeString`**: The workhorse for text inputs - names, descriptions, URLs
**`ParameterTypeSecret`**: Like strings, but values are masked in help and logs (perfect for passwords, API keys)
**`ParameterTypeInteger`**: Whole numbers with automatic range validation
**`ParameterTypeFloat`**: Decimal numbers for measurements, percentages, ratios
**`ParameterTypeBool`**: True/false flags that work with `--flag` and `--no-flag` patterns
**`ParameterTypeDate`**: Intelligent date parsing that handles multiple formats

#### Collection Types
**`ParameterTypeStringList`**: Multiple values like `--tag web --tag api --tag production`
**`ParameterTypeIntegerList`**: Lists of numbers for ports, IDs, quantities
**`ParameterTypeFloatList`**: Multiple decimal values for coordinates, measurements

#### Choice Types  
**`ParameterTypeChoice`**: Single selection from predefined options (with tab completion!)
**`ParameterTypeChoiceList`**: Multiple selections from predefined options

#### File Types
**`ParameterTypeFile`**: File paths with existence validation and tab completion
**`ParameterTypeFileList`**: Multiple file paths
**`ParameterTypeStringFromFile`**: Read text content from a file (useful for large inputs)
**`ParameterTypeStringListFromFile`**: Read line-separated lists from files

#### Special Types
**`ParameterTypeKeyValue`**: Map-like inputs: `--env DATABASE_URL=postgres://... --env DEBUG=true`

### Parameter Definition Options

```go
parameters.NewParameterDefinition(
    "parameter-name",                    // Required: parameter name
    parameters.ParameterTypeString,      // Required: parameter type
    
    // Optional configuration
    parameters.WithDefault("default"),   // Default value
    parameters.WithHelp("Description"),  // Help text
    parameters.WithRequired(true),       // Mark as required
    parameters.WithShortFlag("n"),       // Short flag (-n)
    
    // For choice types
    parameters.WithChoices("opt1", "opt2", "opt3"),
    
    // For file types
    parameters.WithFileExtensions(".txt", ".md"),
)
```

### Working with Arguments

Arguments are positional parameters that don't use flags:

```go
cmds.WithArguments(
    parameters.NewParameterDefinition(
        "input-file",
        parameters.ParameterTypeFile,
        parameters.WithHelp("Input file to process"),
        parameters.WithRequired(true),
    ),
    parameters.NewParameterDefinition(
        "output-file",
        parameters.ParameterTypeString,
        parameters.WithHelp("Output file path"),
        parameters.WithRequired(false),
    ),
)
```

## Command Building and Registration

### Integration with Cobra

Glazed provides several ways to convert commands to Cobra commands:

#### Option A: Automatic Builder (Recommended)
```go
// Automatically selects the appropriate builder based on command type
cobraCmd, err := cli.BuildCobraCommandFromCommand(myCmd)
if err != nil {
    log.Fatalf("Error building Cobra command: %v", err)
}
```

#### Option B: Specific Builders
```go
// Use specific builders for more control
if glazeCmd, ok := myCmd.(cmds.GlazeCommand); ok {
    cobraCmd, err = cli.BuildCobraCommandFromGlazeCommand(glazeCmd)
} else if writerCmd, ok := myCmd.(cmds.WriterCommand); ok {
    cobraCmd, err = cli.BuildCobraCommandFromWriterCommand(writerCmd)
} else if bareCmd, ok := myCmd.(cmds.BareCommand); ok {
    cobraCmd, err = cli.BuildCobraCommandFromBareCommand(bareCmd)
}
```

#### Option C: Dual Command Builder (For Dual Commands)
```go
// For commands that implement multiple interfaces
cobraCmd, err := cli.BuildCobraCommandDualMode(
    myCmd,
    cli.WithGlazeToggleFlag("with-glaze-output"),
)
if err != nil {
    log.Fatalf("Error building Cobra command: %v", err)
}
```

### Dual Command Builder Options

The dual command builder supports several customization options:

```go
cobraCmd, err := cli.BuildCobraCommandDualMode(
    dualCmd,
    // Customize the toggle flag name
    cli.WithGlazeToggleFlag("structured-output"),
    
    // Hide specific glaze flags even when in glaze mode
    cli.WithHiddenGlazeFlags("template", "select"),
    
    // Make glaze mode the default (adds --no-glaze-output flag instead)
    cli.WithDefaultToGlaze(),
)
```

## Running Commands

### Programmatic Execution

To run a command programmatically without Cobra:

```go
// Create command instance
cmd, err := NewMyCommand()
if err != nil {
    log.Fatalf("Error creating command: %v", err)
}

// Set up execution context
ctx := context.Background()

// Define parameter values
parseOptions := []runner.ParseOption{
    runner.WithValuesForLayers(map[string]map[string]interface{}{
        "default": {
            "count": 20,
            "format": "json",
        },
    }),
    runner.WithEnvMiddleware("MYAPP_"),
}

// Configure output
runOptions := []runner.RunOption{
    runner.WithWriter(os.Stdout),
}

// Run the command
err = runner.ParseAndRun(ctx, cmd, parseOptions, runOptions)
if err != nil {
    log.Fatalf("Error running command: %v", err)
}
```

### Parameter Loading Sources

Parameters can be loaded from multiple sources (in priority order):

1. **Command line arguments** (highest priority)
2. **Environment variables** 
3. **Configuration files**
4. **Default values** (lowest priority)

```go
parseOptions := []runner.ParseOption{
    // Load from environment with prefix
    runner.WithEnvMiddleware("MYAPP_"),
    
    // Load from configuration file
    runner.WithViper(),
    
    // Set explicit values
    runner.WithValuesForLayers(map[string]map[string]interface{}{
        "default": {"count": 10},
    }),
    
    // Add custom middleware
    runner.WithAdditionalMiddlewares(customMiddleware),
}
```

## Structured Data Output (GlazeCommand)

### Creating Rows

For GlazeCommands, rows form the structured output:

#### Using NewRow with MRP (MapRowPair)
```go
row := types.NewRow(
    types.MRP("id", 1),
    types.MRP("name", "John Doe"),
    types.MRP("email", "john@example.com"),
    types.MRP("active", true),
)
```

#### From a map
```go
data := map[string]interface{}{
    "id":     1,
    "name":   "John Doe",
    "email":  "john@example.com",
    "active": true,
}
row := types.NewRowFromMap(data)
```

#### From a struct
```go
type User struct {
    ID     int    `json:"id"`
    Name   string `json:"name"`
    Email  string `json:"email"`
    Active bool   `json:"active"`
}

user := User{ID: 1, Name: "John Doe", Email: "john@example.com", Active: true}
row := types.NewRowFromStruct(&user, true) // true = lowercase field names
```

### Adding Rows to Processor

```go
func (c *MyCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    // Process data and create rows
    for _, item := range data {
        row := types.NewRow(
            types.MRP("field1", item.Value1),
            types.MRP("field2", item.Value2),
        )
        
        if err := gp.AddRow(ctx, row); err != nil {
            return err
        }
    }
    
    return nil
}
```

## Advanced Patterns

### Conditional Interface Implementation

Commands can implement multiple interfaces and choose behavior at runtime:

```go
type FlexibleCommand struct {
    *cmds.CommandDescription
    mode string // "simple" or "structured"
}

func (c *FlexibleCommand) Run(ctx context.Context, parsedLayers *layers.ParsedLayers) error {
    // Simple text output
    fmt.Println("Simple output mode")
    return nil
}

func (c *FlexibleCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    // Structured output
    row := types.NewRow(types.MRP("mode", "structured"))
    return gp.AddRow(ctx, row)
}

// Interface assertions
var _ cmds.BareCommand = &FlexibleCommand{}
var _ cmds.GlazeCommand = &FlexibleCommand{}
```

### Command Composition

Create commands that delegate to other commands:

```go
type CompositeCommand struct {
    *cmds.CommandDescription
    subCommands []cmds.Command
}

func (c *CompositeCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    for _, subCmd := range c.subCommands {
        if glazeCmd, ok := subCmd.(cmds.GlazeCommand); ok {
            if err := glazeCmd.RunIntoGlazeProcessor(ctx, parsedLayers, gp); err != nil {
                return err
            }
        }
    }
    return nil
}
```

### Dynamic Command Generation

Generate commands at runtime based on configuration:

```go
func CreateCommandsFromConfig(config *Config) ([]cmds.Command, error) {
    var commands []cmds.Command
    
    for _, cmdConfig := range config.Commands {
        cmd, err := createCommandFromConfig(cmdConfig)
        if err != nil {
            return nil, err
        }
        commands = append(commands, cmd)
    }
    
    return commands, nil
}

func createCommandFromConfig(config CommandConfig) (cmds.Command, error) {
    // Create layers based on config
    var layers []layers.ParameterLayer
    
    for _, layerConfig := range config.Layers {
        layer, err := createLayerFromConfig(layerConfig)
        if err != nil {
            return nil, err
        }
        layers = append(layers, layer)
    }
    
    // Create command description
    cmdDesc := cmds.NewCommandDescription(
        config.Name,
        cmds.WithShort(config.Short),
        cmds.WithLong(config.Long),
        cmds.WithLayersList(layers...),
    )
    
    // Return appropriate command type based on config
    switch config.Type {
    case "bare":
        return &DynamicBareCommand{CommandDescription: cmdDesc}, nil
    case "glaze":
        return &DynamicGlazeCommand{CommandDescription: cmdDesc}, nil
    default:
        return nil, fmt.Errorf("unknown command type: %s", config.Type)
    }
}
```

## Error Handling Patterns

### Graceful Error Handling

```go
func (c *MyCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    s := &MyCommandSettings{}
    if err := parsedLayers.InitializeStruct(layers.DefaultSlug, s); err != nil {
        return fmt.Errorf("failed to initialize settings: %w", err)
    }
    
    // Validate settings
    if err := c.validateSettings(s); err != nil {
        return fmt.Errorf("invalid settings: %w", err)
    }
    
    // Process with context cancellation support
    for i := 0; i < s.Count; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            row := types.NewRow(types.MRP("id", i))
            if err := gp.AddRow(ctx, row); err != nil {
                return fmt.Errorf("failed to add row %d: %w", i, err)
            }
        }
    }
    
    return nil
}

func (c *MyCommand) validateSettings(s *MyCommandSettings) error {
    if s.Count < 0 {
        return errors.New("count must be non-negative")
    }
    if s.Count > 1000 {
        return errors.New("count cannot exceed 1000")
    }
    return nil
}
```

### Exit Control

Commands can control application exit behavior:

```go
func (c *MyCommand) Run(ctx context.Context, parsedLayers *layers.ParsedLayers) error {
    // For early exit without error
    if shouldExit {
        return &cmds.ExitWithoutGlazeError{}
    }
    
    // Normal processing
    return nil
}
```

## Performance Considerations

### Efficient Row Creation

For large datasets, optimize row creation:

```go
func (c *MyCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    // Pre-allocate row slice for better performance
    const batchSize = 1000
    
    for batch := 0; batch < totalBatches; batch++ {
        // Process in batches to avoid memory issues
        batchData := getDataBatch(batch, batchSize)
        
        for _, item := range batchData {
            // Reuse row objects when possible
            row := types.NewRow(
                types.MRP("id", item.ID),
                types.MRP("value", item.Value),
            )
            
            if err := gp.AddRow(ctx, row); err != nil {
                return err
            }
        }
        
        // Check for cancellation between batches
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
    }
    
    return nil
}
```

### Memory Management

For commands processing large amounts of data:

```go
func (c *LargeDataCommand) RunIntoGlazeProcessor(
    ctx context.Context,
    parsedLayers *layers.ParsedLayers,
    gp middlewares.Processor,
) error {
    // Use streaming approach instead of loading all data at once
    reader, err := openDataReader()
    if err != nil {
        return err
    }
    defer reader.Close()
    
    scanner := bufio.NewScanner(reader)
    for scanner.Scan() {
        // Process one line at a time
        line := scanner.Text()
        data, err := parseLineToData(line)
        if err != nil {
            continue // Skip invalid lines
        }
        
        row := types.NewRowFromStruct(data, true)
        if err := gp.AddRow(ctx, row); err != nil {
            return err
        }
    }
    
    return scanner.Err()
}
```

## Best Practices

### Interface Selection

Choose interfaces based on user requirements:

- **BareCommand** when users need rich feedback, progress updates, or interactive elements
- **WriterCommand** when output might go to files, logs, or other destinations  
- **GlazeCommand** when data will be processed, filtered, or integrated with other tools
- **Dual Commands** when usage patterns vary

Example: A backup command might start as BareCommand for user feedback (`Backing up 1,247 files...`), but users eventually want structured output for monitoring scripts. A dual command serves both needs.

### Type Safety

Use settings structs with `glazed.parameter` tags to prevent type conversion errors:

```go
// Good: Type-safe and clear
type BackupSettings struct {
    Source      string        `glazed.parameter:"source"`
    Destination string        `glazed.parameter:"destination"`
    MaxAge      time.Duration `glazed.parameter:"max-age"`
    DryRun      bool          `glazed.parameter:"dry-run"`
}

// Avoid: Manual parameter extraction
source, _ := parsedLayers.GetString("source")
maxAge, _ := parsedLayers.GetString("max-age") // Bug waiting to happen!
```

### Defaults and Help

Provide sensible defaults so commands work with minimal flags. If your command requires multiple flags to be useful, reconsider the design.

Write clear help text with examples for complex parameters:

```go
parameters.NewParameterDefinition(
    "filter",
    parameters.ParameterTypeString,
    parameters.WithHelp("Filter results using SQL-like syntax. Examples: 'status = \"active\"', 'created_at > \"2023-01-01\"'"),
)
```

### Error Handling

Provide specific, actionable error messages:

```go
// Good: Specific and actionable
if s.Port < 1 || s.Port > 65535 {
    return fmt.Errorf("port %d is invalid; must be between 1 and 65535", s.Port)
}

// Poor: Vague and frustrating
if !isValidPort(s.Port) {
    return errors.New("invalid port")
}
```

Validate parameters before expensive operations. Always check context cancellation in loops and long operations.

### Performance

Design for streaming to handle large datasets:

```go
// Good: Processes data as it arrives
scanner := bufio.NewScanner(reader)
for scanner.Scan() {
    row := processLine(scanner.Text())
    if err := gp.AddRow(ctx, row); err != nil {
        return err
    }
}

// Problematic: Loads everything into memory first
allData := loadAllDataIntoMemory() // What if this is 10GB?
for _, item := range allData {
    // Process items...
}
```

### Documentation

Command help text is often the primary documentation users read:

- Include realistic examples in the command description
- Explain the purpose, not just the syntax
- Document any side effects or special requirements
- Cross-reference related commands

For GlazeCommands, document the output schema. Users building scripts need to know what fields are available and what they contain.

## Command Evolution

Commands typically evolve through these stages:

1. **MVP**: BareCommand that solves immediate problem
2. **Growth**: Users want to pipe output to other tools → add GlazeCommand interface
3. **Scale**: Command becomes popular, performance matters → add streaming and batching
4. **Maturity**: Users have diverse needs → convert to dual command with rich options

Design for this evolution by keeping business logic separate from output formatting.

### Design Principles

Sometimes simple is better than powerful. If your command has a single, specific purpose and will never need structured output, BareCommand is perfectly fine. Don't add complexity for theoretical future needs.

Use command composition over command complication. Instead of one command that does everything, consider multiple focused commands that work well together.

## Next Steps

1. Start with the [Commands Tutorial](../tutorials/build-first-command.md) for hands-on experience
2. Study the [Layer Guide](layers-guide.md) to understand parameter organization  
3. Explore the [Applications section](../applications/) for real-world examples
4. Build iteratively - start with something that works, then improve based on actual usage
