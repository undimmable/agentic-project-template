# Expressive Logging for Agentic Debugging

## Introduction

This document outlines best practices for implementing expressive logging and effective unit testing in AI agent systems. These practices are designed to help developers and AI agents quickly identify, diagnose, and resolve issues in complex codebases.

The meta-technique described here emerged from troubleshooting experiences where insufficient logging and non-descriptive error messages made it difficult to narrow down problems. By implementing these guidelines, both human developers and AI agents can more efficiently debug issues and maintain robust systems.

## The Problem: Insufficient Logging and Error Information

When debugging the "too many values to unpack (expected 2)" error in the neural network code, the lack of expressive logging made it challenging to:

1. Identify the exact location of the error
2. Understand the data structures involved
3. Trace the execution path
4. Determine the root cause quickly

The error message provided minimal information, and the tests lacked descriptive assertions that could have pinpointed the issue more precisely.

## Principles of Expressive Logging for Agentic Debugging

### 1. Contextual Error Messages

Error messages should provide sufficient context to understand:
- What operation was being performed
- What inputs were involved (with appropriate truncation for large data)
- What the expected outcome was
- What actually happened

**Bad Example:**
```python
assert len(result) == 2, "Wrong number of values"
```

**Good Example:**
```python
assert len(result) == 2, f"Expected tuple of 2 values but got {len(result)} values: {result}"
```

### 2. Hierarchical Logging Structure

Implement a hierarchical logging structure that:
- Clearly shows the execution path
- Indicates entry and exit points of major functions
- Logs transitions between system components
- Uses consistent indentation or prefixes to show hierarchy

**Example:**
```python
logger.info("ENTER: normalize_to_unit_phase with tensors of shape K=%s, Q=%s", K.shape, Q.shape)
# Function logic here
logger.info("EXIT: normalize_to_unit_phase returning tensors of shape %s and %s", 
            normalized_K.shape, normalized_Q.shape)
```

### 3. Data Shape and Type Validation

For AI systems working with tensors and complex data structures:
- Log shapes, types, and summary statistics of inputs and outputs
- Validate data at component boundaries
- Include dimensionality checks with descriptive error messages

**Example:**
```python
def forward(self, x):
    logger.debug("Input tensor shape: %s, dtype: %s", x.shape, x.dtype)
    if x.dim() != 3:
        raise ValueError(f"Expected 3D tensor [batch, seq_len, features], got shape {x.shape}")
    
    # Processing logic
    
    logger.debug("Output tensor shape: %s, range: [%f, %f]", 
                 output.shape, output.min().item(), output.max().item())
    return output
```

### 4. Traceable Execution Paths

Ensure that logs create a traceable path through the system:
- Use unique identifiers for requests/operations
- Include timestamps for performance analysis
- Log major decision points and their outcomes
- Record the flow between distributed components

**Example:**
```python
def process_request(request_id, data):
    logger.info("[%s] Processing request with %d items", request_id, len(data))
    try:
        result = self.neural_component.process(data)
        logger.info("[%s] Neural processing complete, proceeding to reasoning", request_id)
        final_result = self.reasoning_component.evaluate(result)
        logger.info("[%s] Request processing complete", request_id)
        return final_result
    except Exception as e:
        logger.error("[%s] Failed to process request: %s", request_id, str(e), exc_info=True)
        raise
```

## Unit Testing Best Practices for Agentic Debugging

### 1. Descriptive Test Names and Documentation

Tests should have clear, descriptive names and documentation that explain:
- What functionality is being tested
- What inputs are being used
- What the expected outcome is
- Any edge cases being covered

**Example:**
```python
def test_normalize_to_unit_phase_returns_exactly_two_normalized_tensors():
    """
    Test that normalize_to_unit_phase returns exactly two tensors,
    each normalized to unit length along the last dimension.
    """
    # Test implementation
```

### 2. Explicit Assertions with Descriptive Messages

Every assertion should include a descriptive message that:
- Explains what is being checked
- Provides context for the expected values
- Includes actual values when the assertion fails

**Example:**
```python
def test_phase_attention_forward():
    # Setup test data
    batch_size, seq_len, dim = 2, 3, 4
    x = torch.randn(batch_size, seq_len, dim)
    
    # Create model and run forward pass
    model = PhaseAttention(dim)
    output = model(x)
    
    # Assertions with descriptive messages
    assert output.shape == x.shape, \
        f"Output shape {output.shape} doesn't match input shape {x.shape}"
    
    # Check that output values are within expected range
    assert torch.all(output >= -10) and torch.all(output <= 10), \
        f"Output values outside expected range: min={output.min().item()}, max={output.max().item()}"
```

### 3. Comprehensive Test Coverage

Tests should cover:
- Happy paths (normal operation)
- Edge cases (boundary conditions)
- Error cases (expected exceptions)
- Integration points between components

**Example:**
```python
class TestNormalizeToUnitPhase:
    def test_with_normal_tensors(self):
        # Test with typical inputs
        
    def test_with_zero_tensors(self):
        # Test with tensors containing zeros
        
    def test_with_single_element_tensors(self):
        # Test with minimal tensors
        
    def test_with_large_tensors(self):
        # Test with large dimensions
```

### 4. Isolated and Deterministic Tests

Tests should be:
- Independent of each other
- Deterministic (same inputs produce same outputs)
- Isolated from external dependencies when possible
- Clear about their environmental requirements

**Example:**
```python
@pytest.mark.parametrize("seed", [42, 123, 7])
def test_model_output_is_deterministic(seed):
    """Test that the model produces the same output given the same input and seed."""
    torch.manual_seed(seed)
    
    # Create deterministic input
    x = torch.randn(2, 3, 4)
    
    # Run model twice with same input
    model = PhaseAttention(4)
    output1 = model(x)
    output2 = model(x)
    
    # Verify outputs are identical
    assert torch.allclose(output1, output2), \
        "Model produced different outputs for the same input"
```

## Implementation Guidelines for AI Systems

### 1. Automatic Tensor Inspection

Implement utilities that automatically log tensor properties:

```python
def log_tensor_info(tensor, name="tensor"):
    """Log comprehensive information about a tensor."""
    if not isinstance(tensor, torch.Tensor):
        logger.warning("%s is not a tensor, but %s", name, type(tensor))
        return
        
    info = {
        "shape": tensor.shape,
        "dtype": tensor.dtype,
        "device": tensor.device,
        "min": tensor.min().item() if tensor.numel() > 0 else None,
        "max": tensor.max().item() if tensor.numel() > 0 else None,
        "mean": tensor.mean().item() if tensor.numel() > 0 else None,
        "has_nan": torch.isnan(tensor).any().item(),
        "has_inf": torch.isinf(tensor).any().item()
    }
    
    logger.debug("%s info: %s", name, info)
    return info
```

### 2. Structured Exception Handling

Create custom exceptions with rich context:

```python
class NeuralProcessingError(Exception):
    """Exception raised for errors in neural processing components."""
    
    def __init__(self, message, component=None, tensor_shapes=None, additional_info=None):
        self.component = component
        self.tensor_shapes = tensor_shapes or {}
        self.additional_info = additional_info or {}
        
        # Enhance the message with context
        context = f" in {component}" if component else ""
        shapes = f" Tensor shapes: {tensor_shapes}" if tensor_shapes else ""
        
        super().__init__(f"{message}{context}.{shapes}")
```

### 3. Execution Tracing for Complex Workflows

Implement execution tracing for complex workflows:

```python
class ExecutionTracer:
    def __init__(self, request_id):
        self.request_id = request_id
        self.start_time = time.time()
        self.steps = []
        
    def add_step(self, component, action, status="success", **kwargs):
        """Record a step in the execution path."""
        step = {
            "timestamp": time.time(),
            "elapsed": time.time() - self.start_time,
            "component": component,
            "action": action,
            "status": status,
            **kwargs
        }
        self.steps.append(step)
        
        # Log the step
        logger.info(
            "[%s] %s.%s (%s) - %.2fs elapsed", 
            self.request_id, component, action, status, step["elapsed"]
        )
        
        return self
        
    def get_trace(self):
        """Return the complete execution trace."""
        return {
            "request_id": self.request_id,
            "total_time": time.time() - self.start_time,
            "steps": self.steps
        }
```

## Conclusion

Implementing expressive logging and effective unit testing is crucial for debugging AI agent systems. By following these guidelines, developers can create systems that are easier to debug, maintain, and improve over time.

The meta-technique of comprehensive logging and descriptive error messages serves as a force multiplier for both human developers and AI agents, enabling faster problem resolution and more robust systems.

Remember: The time invested in implementing good logging and testing practices pays dividends when debugging complex issues. What might take hours or days to diagnose with poor logging can often be resolved in minutes with expressive, contextual information.