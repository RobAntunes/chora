# Chora: Comprehensive Language Design Document

## Core Paradigms

Our language combines three fundamental innovations:

1. **Arena-Based Memory Management** with direct memory addressing and scope-based safety
2. **Concept-Modifier System** with dynamic capabilities and state transformations
3. **Runtime Safety Guarantees** that maintain memory safety while enabling flexibility

## Memory System

### Direct Memory Addressing

```rust
arena {
    let x = 42;        // at address 0x1234
    process(0x1234);   // pass raw address
    // x is still accessible here if process kept it in scope
} // if nothing references x, it drops and arena may drop
```

- Pass raw memory addresses between functions
- No need for references or complex ownership rules
- Maximum performance - just passing numbers
- Natural and simple to reason about

### Scope-Based Safety

- Values automatically drop when they go out of scope
- Arenas automatically drop when they become empty
- Safety emerges from natural scope rules
- Impossible to have dangling references

```rust
fn process(addr: usize) {
    use_value(addr);   // direct memory access
    // addr drops with scope
    // if this was the last reference, value drops too
} // if arena empty, it drops
```

### Key Properties

- **Memory Safety**: Can't access dropped values (no references exist)
- **Zero Validation Overhead**: Safety through scope rules, not runtime checks
- **No GC Required**: No reference counting or garbage collection
- **Natural Mental Model**: Matches intuitive understanding of scope

## Data Modeling System

### Concepts as Capability Definitions

```rust
concept Counter {
    // Define expected state structure
    expects state: {
        count: number
    }
    
    // Define behaviors
    increment(self): self.state.count += 1
    decrement(self): self.state.count -= 1
    value(self): self.state.count
}
```

- Concepts define expected state shape and behaviors
- Functions operate on self.state
- Clear contract between concept and implementing entity

### Entities as State Containers

```rust
entity UserProfile {
    state: {
        name: "Alice",
        email: "alice@example.com",
        login_count: 0
    },
    implements: [Nameable, Contactable]
}
```

- Entities hold concrete state data
- Explicitly declare implemented concepts
- Initial state defined at creation time

### Modifiers as Dynamic Capabilities

```rust
// Add modifier with methods
user + Premium {
    on_add(self): {
        self.state.premium_until = now() + 30.days
        self.state.features = ["advanced_search", "priority_support"]
    }
    
    on_remove(self): {
        self.state.premium_until = null
        self.state.features = []
    }
    
    has_feature(self, feature): self.state.features.includes(feature)
}

// Remove modifier
user - Premium
```

- Modifiers can be added/removed at runtime
- State transformations on addition/removal
- Dynamic behavior changes
- Separation of core entity from optional capabilities

### Capability Groups

```rust
modifier_group UserCapabilities {
    includes: [Nameable, Contactable, Authenticatable]
}

// Add all capabilities at once
user + UserCapabilities
```

- Group related modifiers for easier management
- Add/remove multiple capabilities atomically
- Logical organization of related functionality

### Contextual Modifiers

```rust
// Only active within transaction blocks
with Transaction(user) {
    // User has temporary Transaction modifier capabilities
    user.update_balance(new_amount)
}
```

- Temporarily grant capabilities within specific contexts
- Automatic cleanup when context ends
- Safer privileged operations

## Runtime System

### Pattern Matching on Capabilities

```rust
match entity {
    has(Premium + Verified) => {
        // Use both Premium and Verified capabilities
        entity.access_premium_feature()
        entity.show_verification_badge()
    },
    has(Premium) => {
        // Use only Premium capabilities
        entity.access_premium_feature()
    },
    _ => {
        // No matching capabilities
        show_upgrade_options(entity)
    }
}
```

- Check for capability combinations
- Different handling based on available capabilities
- Safe access to capabilities

### Capability Negotiation

```rust
// Entities can negotiate capabilities at runtime
fn process(entity_addr: usize, required_capabilities: [Capability]) {
    let entity = from_addr(entity_addr)
    
    // Runtime capability negotiation
    let missing = entity.missing_capabilities(required_capabilities)
    if !missing.empty() {
        // Entity can dynamically adapt to requirements
        if entity.can_acquire(missing) {
            entity.acquire_capabilities(missing)
            // Continue with processing
        }
    }
}
```

- Dynamic capability acquisition based on need
- Graceful handling of missing capabilities
- Flexible runtime behavior

### Runtime Contexts

```rust
runtime Sandbox {
    // Explicitly grant only specific capabilities
    allow capabilities: [BasicMath, ReadOnlyState]
    
    // Any attempt to use other capabilities fails at runtime
    // with arena safety preventing memory corruption
    execute_untrusted_code(code_entity)
}
```

- Different execution environments with different rules
- Capability-based security model
- Safe execution of untrusted code

### State Machines

```rust
state_machine OrderProcess {
    states: [Created, Paid, Shipped, Delivered, Canceled]
    
    transitions: {
        Created -> [Paid, Canceled],
        Paid -> [Shipped, Canceled],
        Shipped -> [Delivered],
        Delivered -> [],
        Canceled -> []
    }
}

// Runtime enforces valid transitions
order.transition_to(Shipped)  // Succeeds only if current state is Paid
```

- Define valid state transitions
- Runtime enforcement of state machine rules
- Clear modeling of process flows

## Cross-Entity Relationships

```rust
arena {
    let team = entity {
        state: {
            name: "Engineering",
            members: []  // Will store addresses
        },
        implements: [Group]
    }
    
    let alice = entity {
        state: {
            name: "Alice",
            role: "Engineer"
        },
        implements: [Person]
    }
    
    // Establish relationship by storing address
    team.add_member(alice)  // Stores alice's address in team.members
    
    // Reference traversal
    for member_addr in team.members {
        let member = from_addr(member_addr)
        print("Team member: {}", member.name)
    }
}
```

- Relationships modeled by storing addresses
- Memory safety maintained through arena system
- Natural many-to-many, one-to-many relationships

### Distributed Capability Negotiation

```rust
distributed_system {
    // Local entity with local capabilities
    let entity = create_entity()
    
    // Request capabilities from remote system
    let remote_capabilities = request_capabilities(remote_system)
    
    // Apply remotely-provided capabilities locally
    if SecurityValidator.validate(remote_capabilities) {
        entity.add_capabilities(remote_capabilities)
    }
    
    // Use combined local and remote capabilities
    process(entity)
}
```

- Safely distribute capabilities across systems
- Runtime security validation
- Dynamic capability acquisition

## Developer Experience

### Minimizing Mental Overhead

```rust
// Implicit arena creation for top level functions
fn main() {
    // Implicit arena exists here
    
    let counter = Counter {
        count: 0
    }
    
    // Automatically resolves address lookups
    print("Count: {}", counter.count)
    
    // Automatic address resolution for method calls
    counter.increment()
    
    // Explicit when needed
    let addr = address_of(counter)
    process(addr)
}
```

- Implicit arena creation
- Automatic address resolution
- Clean, familiar syntax for common operations
- Explicit operations available when needed

### Safe Introspection

```rust
// Examine entity structure
introspect entity {
    // Safe access to structure information
    print("State fields: {}", entity.state_fields())
    print("Implemented concepts: {}", entity.concepts())
    print("Available modifiers: {}", entity.modifiers())
    
    // Can manipulate based on introspection
    for field in entity.state_fields() {
        if field.type == "string" {
            entity.state[field.name] = entity.state[field.name].to_uppercase()
        }
    }
}
```

- Safe reflection capabilities
- Dynamic code based on entity structure
- Maintains memory safety guarantees

## Distinguishing Features From Existing Languages

### vs. Rust Traits

- **Dynamic Addition/Removal**: Unlike Rust's static traits, our modifiers can be added/removed at runtime
- **State Transformation**: Modifiers can transform entity state when added/removed
- **No Orphan Rules**: Modifiers can be added to any entity regardless of origin
- **Simpler Syntax**: More concise capability management

### vs. Object-Oriented Programming

- **No Inheritance Hierarchy**: Composition over inheritance
- **Dynamic Capabilities**: Add/remove behavior at runtime
- **Direct Memory Model**: No reference indirection
- **Pattern Matching**: First-class concept for capability checks

### vs. Functional Programming

- **Mutable State**: Explicit state management
- **Identity Preservation**: Entities maintain identity
- **Capability-Based**: Behavior through explicit capabilities
- **Arena Memory**: No garbage collection overhead

## Future Considerations

The following areas will need to be developed as we move forward:

1. **Type System**: Formalization of type compatibility, conversion, and generics
2. **Error Handling**: Mechanism for dealing with capability errors and state transitions
3. **Concurrency Model**: Integration with arena memory system
4. **Module System**: Organization of concepts, modifiers, and entities
5. **Standard Library**: Core concepts and modifiers provided by default

## Conclusion

Our language represents a radical departure from current paradigms by combining arena-based memory safety with dynamic capabilities and runtime guarantees. This unique combination enables new programming patterns and systems that are difficult or impossible to express in existing languages, while maintaining strong safety guarantees and an intuitive programming model.
