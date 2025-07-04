## Writing Readable, Maintainable Code

### Core Philosophy

Write code as if the person maintaining it is a violent psychopath who knows where you live. More practically:

1. **Optimize for readability by default**. Performance optimizations only when explicitly requested.
2. **Code for teams**: Assume multiple developers will read, modify, and debug this code.
3. **Progressive complexity**: Start simple, add complexity only when needed.
4. **Explicit is better than implicit**: Make intentions clear, avoid magic.

### Universal Principles

#### 1. Naming Conventions
```python
# BAD: Ambiguous, abbreviated
def proc_usr_data(u, t):
    return u.get_info() if t > 0 else None

# GOOD: Self-documenting
def fetch_user_profile(user_id: str, timeout_seconds: int) -> Optional[UserProfile]:
    if timeout_seconds <= 0:
        return None
    return user_service.get_profile(user_id)
```

#### 2. Function Design - Start Simple, Scale Up

**Simple Flow Example**:
```typescript
// Level 1: Basic validation
function isValidEmail(email: string): boolean {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
}

// Level 2: Add context and error details
interface ValidationResult {
    isValid: boolean;
    errors: string[];
}

function validateEmail(email: string): ValidationResult {
    const errors: string[] = [];
    
    if (!email) {
        errors.push("Email is required");
    } else if (!email.includes("@")) {
        errors.push("Email must contain @ symbol");
    } else if (!isValidEmailFormat(email)) {
        errors.push("Invalid email format");
    }
    
    return {
        isValid: errors.length === 0,
        errors
    };
}

// Level 3: Complex business logic with multiple validators
class EmailValidator {
    private validators: EmailValidationRule[] = [
        new FormatValidator(),
        new DomainWhitelistValidator(allowedDomains),
        new DisposableEmailValidator(),
        new MXRecordValidator()
    ];
    
    async validate(email: string): Promise<ValidationResult> {
        const results = await Promise.all(
            this.validators.map(v => v.validate(email))
        );
        
        return ValidationResult.merge(results);
    }
}
```

### Domain-Specific Patterns

#### Web Services
```go
// BAD: Everything in one handler
func userHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == "GET" {
        // 50 lines of GET logic
    } else if r.Method == "POST" {
        // 80 lines of POST logic
    }
    // More methods...
}

// GOOD: Separation of concerns
type UserHandler struct {
    userService UserService
    validator   Validator
    logger      Logger
}

func (h *UserHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        h.handleGet(w, r)
    case http.MethodPost:
        h.handleCreate(w, r)
    default:
        h.respondError(w, http.StatusMethodNotAllowed, "Method not allowed")
    }
}

func (h *UserHandler) handleGet(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := chi.URLParam(r, "userID")
    
    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        h.handleServiceError(w, err)
        return
    }
    
    h.respondJSON(w, http.StatusOK, user)
}

// Reusable response helpers
func (h *UserHandler) respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    
    if err := json.NewEncoder(w).Encode(data); err != nil {
        h.logger.Error("Failed to encode response", "error", err)
    }
}
```

#### Cloud Technologies (AWS/GCP/Azure)
```python
# BAD: Hardcoded, no error handling, no retry
import boto3

def upload_file(file_path):
    s3 = boto3.client('s3')
    s3.upload_file(file_path, 'my-bucket', 'file.txt')

# GOOD: Configurable, resilient, observable
import boto3
from botocore.exceptions import ClientError
from typing import Optional
import logging
from tenacity import retry, stop_after_attempt, wait_exponential

class S3FileUploader:
    def __init__(self, 
                 bucket_name: str,
                 region: str = 'us-east-1',
                 logger: Optional[logging.Logger] = None):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3', region_name=region)
        self.logger = logger or logging.getLogger(__name__)
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10)
    )
    def upload_file(self, 
                    local_path: str, 
                    s3_key: str,
                    metadata: Optional[dict] = None) -> str:
        """
        Upload file to S3 with automatic retry and error handling.
        
        Returns:
            S3 object URL on success
        
        Raises:
            S3UploadError: If upload fails after retries
        """
        try:
            extra_args = {}
            if metadata:
                extra_args['Metadata'] = metadata
            
            self.logger.info(f"Uploading {local_path} to s3://{self.bucket_name}/{s3_key}")
            
            self.s3_client.upload_file(
                Filename=local_path,
                Bucket=self.bucket_name,
                Key=s3_key,
                ExtraArgs=extra_args
            )
            
            url = f"https://{self.bucket_name}.s3.amazonaws.com/{s3_key}"
            self.logger.info(f"Successfully uploaded to {url}")
            return url
            
        except ClientError as e:
            error_code = e.response['Error']['Code']
            self.logger.error(f"S3 upload failed: {error_code}", exc_info=True)
            raise S3UploadError(f"Failed to upload to S3: {error_code}") from e
```

### Complexity Scaling Patterns

#### 1. Configuration Management
```typescript
// Simple: Direct configuration
const API_URL = "https://api.example.com";

// Medium: Environment-based
interface Config {
    apiUrl: string;
    timeout: number;
    retryAttempts: number;
}

const config: Config = {
    apiUrl: process.env.API_URL || "https://api.example.com",
    timeout: parseInt(process.env.TIMEOUT || "5000"),
    retryAttempts: parseInt(process.env.RETRY_ATTEMPTS || "3")
};

// Complex: Full configuration system
class ConfigurationManager {
    private static instance: ConfigurationManager;
    private config: Map<string, any> = new Map();
    
    static getInstance(): ConfigurationManager {
        if (!this.instance) {
            this.instance = new ConfigurationManager();
        }
        return this.instance;
    }
    
    loadFromEnvironment(): void {
        this.loadDefaults();
        this.overrideFromEnv();
        this.validateConfig();
    }
    
    get<T>(key: string, defaultValue?: T): T {
        if (!this.config.has(key) && defaultValue === undefined) {
            throw new ConfigurationError(`Missing required configuration: ${key}`);
        }
        return this.config.get(key) ?? defaultValue;
    }
}
```

#### 2. Error Handling Evolution
```rust
// Simple: Basic Result
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// Medium: Custom error types
#[derive(Debug)]
enum MathError {
    DivisionByZero,
    Overflow,
    InvalidInput(String),
}

fn divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        Err(MathError::DivisionByZero)
    } else {
        Ok(a / b)
    }
}

// Complex: Full error handling with context
use thiserror::Error;

#[derive(Error, Debug)]
pub enum CalculationError {
    #[error("Division by zero attempted with numerator {numerator}")]
    DivisionByZero { numerator: f64 },
    
    #[error("Calculation overflow: {expression}")]
    Overflow { expression: String },
    
    #[error("Invalid input: {message}")]
    InvalidInput { message: String, source: Box<dyn std::error::Error> },
}

pub struct Calculator {
    precision: u32,
    allow_infinity: bool,
}

impl Calculator {
    pub fn divide(&self, a: f64, b: f64) -> Result<f64, CalculationError> {
        if b == 0.0 {
            if self.allow_infinity {
                Ok(f64::INFINITY)
            } else {
                Err(CalculationError::DivisionByZero { numerator: a })
            }
        } else {
            Ok((a / b).round_to(self.precision))
        }
    }
}
```

### Testing Patterns

```python
# BAD: Untestable code
class EmailService:
    def send_welcome_email(self, user_id):
        user = database.query(f"SELECT * FROM users WHERE id = {user_id}")
        smtp = smtplib.SMTP('smtp.gmail.com', 587)
        smtp.send_message(f"Welcome {user.name}!")

# GOOD: Testable with dependency injection
class EmailService:
    def __init__(self, user_repository: UserRepository, 
                 email_sender: EmailSender):
        self.user_repository = user_repository
        self.email_sender = email_sender
    
    async def send_welcome_email(self, user_id: str) -> None:
        user = await self.user_repository.get_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        message = WelcomeEmailTemplate.render(user=user)
        await self.email_sender.send(
            to=user.email,
            subject="Welcome to our platform!",
            body=message
        )

# Easy to test:
async def test_send_welcome_email():
    mock_repo = Mock(UserRepository)
    mock_sender = Mock(EmailSender)
    service = EmailService(mock_repo, mock_sender)
    
    mock_repo.get_by_id.return_value = User(id="123", name="Test")
    
    await service.send_welcome_email("123")
    
    mock_sender.send.assert_called_once()
```

### Performance Code (Only When Requested)

```go
// When asked for performance optimization:
// "Make this CSV parser faster for large files"

// Performance-focused version with comments explaining trade-offs
func ParseLargeCSV(filename string) ([][]string, error) {
    // Pre-allocate with estimated capacity to avoid reallocations
    // Trade-off: Uses more memory upfront
    estimatedRows := 1_000_000
    results := make([][]string, 0, estimatedRows)
    
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer file.Close()
    
    // Use buffered reader for better I/O performance
    // Buffer size tuned for typical L2 cache size
    reader := bufio.NewReaderSize(file, 256*1024)
    
    // Reuse the same slice for each line to reduce allocations
    // Trade-off: More complex code, harder to maintain
    lineBuffer := make([]byte, 0, 4096)
    
    for {
        lineBuffer = lineBuffer[:0] // Reset buffer
        
        // Read line into reusable buffer
        line, isPrefix, err := reader.ReadLine()
        if err == io.EOF {
            break
        }
        if err != nil {
            return nil, err
        }
        
        lineBuffer = append(lineBuffer, line...)
        
        // Handle lines longer than buffer
        for isPrefix {
            line, isPrefix, err = reader.ReadLine()
            if err != nil {
                return nil, err
            }
            lineBuffer = append(lineBuffer, line...)
        }
        
        // Parse CSV line manually for speed
        // Trade-off: Doesn't handle all CSV edge cases
        fields := fastSplitCSV(lineBuffer)
        results = append(results, fields)
    }
    
    return results, nil
}

// Comment explaining why this is faster but less maintainable
// than using encoding/csv
```

### Code Review Mental Model

Before submitting code, ask:

1. **Can a junior developer understand this without explanation?**
2. **Can this code be unit tested without modifying it?**
3. **If this breaks at 3 AM, are the error messages helpful?**
4. **Does each function have a single, clear purpose?**
5. **Are edge cases handled explicitly?**

### Common Pitfalls to Avoid

1. **Premature abstraction**: Don't create interfaces until you have 2+ implementations
2. **Clever one-liners**: If it needs a comment to explain, it's too clever
3. **Deep nesting**: More than 3 levels usually means extract a function
4. **Giant functions**: If it doesn't fit on one screen, split it
5. **Ignoring errors**: Always handle or explicitly propagate

Remember: Code is written once but read many times. Optimize for the reader, not the writer.
