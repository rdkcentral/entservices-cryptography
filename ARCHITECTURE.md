# Cryptography Plugin - Architecture Documentation

## Overview

The Cryptography plugin is a WPEFramework (Thunder) plugin that provides secure cryptographic operations through an external access interface. It enables applications to perform cryptographic functions such as encryption, decryption, hashing, and secure key management in a controlled and isolated environment.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│           WPEFramework (Thunder) Core                   │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │  CryptographyExtAccess │  (Plugin Entry Point)
         │      (IPlugin)         │
         └───────────┬────────────┘
                     │ manages
         ┌───────────▼─────────────────┐
         │ CryptographyImplementation  │  (IConfiguration)
         │                             │
         │  ┌─────────────────────┐   │
         │  │  ExternalAccess     │   │  (RPC Communicator)
         │  │  (UNIX Socket)      │   │
         │  └─────────┬───────────┘   │
         └────────────┼─────────────────┘
                      │
         ┌────────────▼───────────────┐
         │   ICryptography Interface  │  (Thunder Framework)
         │   (Core Crypto Library)    │
         └────────────────────────────┘
```

## Component Architecture

### 1. CryptographyExtAccess (Plugin Interface)

**Purpose**: Main plugin entry point that manages the plugin lifecycle within WPEFramework.

**Key Responsibilities**:
- Plugin initialization and deinitialization
- Managing remote procedure call (RPC) connections
- Monitoring connection health through notifications
- Interfacing with Thunder's plugin host system

**Dependencies**:
- `PluginHost::IShell`: Thunder plugin shell interface
- `Exchange::IConfiguration`: Configuration management interface
- `RPC::IRemoteConnection`: Remote connection management

**Lifecycle Management**:
- `Initialize()`: Establishes connection to CryptographyImplementation via RPC
- `Deinitialize()`: Cleanly tears down RPC connections and releases resources
- `Deactivated()`: Handles unexpected connection terminations

### 2. CryptographyImplementation (Service Core)

**Purpose**: Core implementation that bridges external access with the Thunder cryptography library.

**Key Responsibilities**:
- Creating and managing the RPC communication channel
- Instantiating the Thunder cryptography interface
- Configuring external access permissions
- Managing the lifecycle of cryptographic services

**Configuration**:
- Connector path: `/tmp/icryptography` (UNIX domain socket)
- Access control: Configured for "apps" group with read/write permissions
- Uses WPEFramework's configuration system for runtime settings

### 3. ExternalAccess (RPC Layer)

**Purpose**: Provides secure inter-process communication for cryptographic operations.

**Architecture**:
- Extends `RPC::Communicator` for Thunder RPC support
- UNIX domain socket communication at `/tmp/icryptography`
- Interface versioning support (version 1)
- Managed by `InvokeServer` for request/response handling

**Security Features**:
- Socket file permissions: Group read/write access
- Group ownership: "apps" group
- Isolated from main process for enhanced security

## Data Flow

### Plugin Initialization Flow

1. WPEFramework loads CryptographyExtAccess plugin
2. `Initialize()` called with plugin configuration
3. Plugin registers connection notification handlers
4. Creates RPC connection to CryptographyImplementation
5. CryptographyImplementation instantiates ICryptography interface
6. ExternalAccess creates UNIX socket at `/tmp/icryptography`
7. Socket permissions configured for external application access
8. Plugin reports ready state to framework

### Cryptographic Operation Flow

1. External application connects to `/tmp/icryptography` socket
2. Request routed through ExternalAccess RPC communicator
3. ExternalAccess validates interface version and acquires interface
4. Request forwarded to ICryptography implementation
5. Cryptographic operation performed by Thunder library
6. Result returned through RPC channel to application
7. Connection maintained for subsequent operations

## Integration with Thunder Framework

### Plugin Framework Integration

- **Plugin Type**: Out-of-process (runs in separate process for security)
- **Activation Mode**: Configurable (Local/Off/Container)
- **Subsystem Dependencies**: PLATFORM subsystem required
- **Interface Aggregation**: Aggregates ICryptography interface

### Build System Integration

- Uses CMake with WPEFramework package discovery
- Links against Thunder core libraries:
  - `${NAMESPACE}Plugins`: Plugin infrastructure
  - `${NAMESPACE}Definitions`: Common definitions
  - `${NAMESPACE}Cryptography`: Cryptography implementation
- Generates plugin configuration files automatically

### Configuration System

Plugin configuration managed through JSON configuration file:
- `CryptographyExtAccess.config`: Plugin-specific settings
- Supports runtime configuration via `ConfigLine()`
- Connector path customizable via configuration

## Security Architecture

### Process Isolation

The plugin runs as a separate process from the main WPEFramework core, providing:
- Memory isolation between cryptographic operations and main framework
- Crash isolation - crypto process failures don't affect main system
- Enhanced security boundary for sensitive operations

### Access Control

- UNIX socket-based communication with filesystem permissions
- Group-based access control (apps group)
- Interface versioning prevents incompatible client access
- RPC layer provides additional request validation

### Resource Management

- Automatic cleanup on connection failure
- Reference counting for interface management
- Graceful shutdown procedures to prevent resource leaks
- Connection monitoring via notification system

## Technical Dependencies

### Core Dependencies

- **WPEFramework Core**: Thunder plugin infrastructure
- **WPEFramework Cryptography**: Core cryptography library implementation
- **RPC Framework**: Thunder's remote procedure call system
- **CompileSettingsDebug**: Debug compilation settings

### System Dependencies

- POSIX system calls (stat, chmod, chown)
- Group management (getgrnam)
- UNIX domain sockets
- Standard C++ libraries (C++11)

## Extension Points

### Adding New Cryptographic Operations

New cryptographic operations can be added by:
1. Extending the `Exchange::ICryptography` interface definition
2. Implementing new methods in cryptography library
3. No changes required in plugin code due to interface aggregation

### Custom Connector Configuration

Alternative communication channels can be configured:
1. Modify connector path in configuration JSON
2. Adjust permissions as needed for security requirements
3. Plugin automatically adapts to configuration

## Performance Considerations

- **RPC Overhead**: UNIX socket communication introduces minimal latency
- **Process Boundary**: Context switches between processes add overhead
- **Connection Pooling**: Single connection per client maintained for efficiency
- **Interface Caching**: RPC interface references cached to avoid repeated lookups

## Monitoring and Diagnostics

- Connection state tracked via notification callbacks
- Trace logging at Information level for key operations
- Error logging for connection failures
- Connection ID tracking for debugging
