# Day 23: Code Quality & Testing - Best Practices and Tools

## 📚 Learning Objectives
- Implement static code analysis tools (SonarQube, ESLint, Pylint)
- Master unit and integration testing frameworks
- Understand code coverage and quality gates
- Practice automated testing strategies
- Set up quality metrics and monitoring

## 🎯 Code Quality Overview

### What is Code Quality?
Code quality refers to how well-written, maintainable, readable, and efficient code is. It encompasses various aspects including functionality, reliability, usability, efficiency, maintainability, and portability.

### Key Quality Metrics
- **Code Coverage**: Percentage of code tested
- **Cyclomatic Complexity**: Measure of code complexity
- **Duplication**: Amount of duplicated code
- **Maintainability Index**: Overall maintainability score
- **Technical Debt**: Effort needed to fix quality issues

## 🛠️ Static Code Analysis Tools

### SonarQube Setup and Configuration

#### 1.1 Install SonarQube with Docker
```bash
# Create SonarQube directory
mkdir sonarqube-setup
cd sonarqube-setup

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_logs:/opt/sonarqube/logs
    ports:
      - "9000:9000"
    
  db:
    image: postgres:13
    container_name: sonarqube-db
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_logs:
  postgresql_data:
EOF

# Start SonarQube
docker-compose up -d

# Wait for startup (check logs)
docker-compose logs -f sonarqube

# Access SonarQube at http://localhost:9000
# Default credentials: admin/admin
```

#### 1.2 Configure SonarQube Project
```bash
# Create sonar-project.properties
cat > sonar-project.properties << 'EOF'
# Project identification
sonar.projectKey=devops-demo-project
sonar.projectName=DevOps Demo Project
sonar.projectVersion=1.0

# Source code location
sonar.sources=src
sonar.tests=tests

# Language-specific settings
sonar.python.coverage.reportPaths=coverage.xml
sonar.javascript.lcov.reportPaths=coverage/lcov.info

# Exclusions
sonar.exclusions=**/node_modules/**,**/vendor/**,**/*.min.js
sonar.test.exclusions=**/tests/**

# Quality gate settings
sonar.qualitygate.wait=true
EOF
```

### Python Code Quality Tools

#### 2.1 Pylint Configuration
```bash
# Install Pylint
pip install pylint

# Create .pylintrc configuration
cat > .pylintrc << 'EOF'
[MASTER]
init-hook='import sys; sys.path.append("src")'

[MESSAGES CONTROL]
disable=C0114,C0115,C0116,R0903

[FORMAT]
max-line-length=88
good-names=i,j,k,ex,Run,_,id,db

[DESIGN]
max-args=7
max-locals=15
max-returns=6
max-branches=12
max-statements=50

[SIMILARITIES]
min-similarity-lines=4
ignore-comments=yes
ignore-docstrings=yes
EOF

# Run Pylint
pylint src/backend/app.py --output-format=json > pylint-report.json
```

#### 2.2 Black Code Formatter
```bash
# Install Black
pip install black

# Create pyproject.toml
cat > pyproject.toml << 'EOF'
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | _build
  | buck-out
  | build
  | dist
)/
'''
EOF

# Format code
black src/backend/
```

#### 2.3 pytest Configuration
```bash
# Install pytest and plugins
pip install pytest pytest-cov pytest-html pytest-xdist

# Create pytest.ini
cat > pytest.ini << 'EOF'
[tool:pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
addopts = 
    --strict-markers
    --strict-config
    --verbose
    --cov=src
    --cov-report=html
    --cov-report=xml
    --cov-report=term-missing
    --html=reports/pytest-report.html
    --self-contained-html
markers =
    unit: Unit tests
    integration: Integration tests
    slow: Slow running tests
EOF
```

### JavaScript/Node.js Code Quality

#### 3.1 ESLint Configuration
```bash
# Initialize Node.js project
cd src/frontend
npm init -y

# Install ESLint
npm install --save-dev eslint @eslint/js

# Create eslint.config.js
cat > eslint.config.js << 'EOF'
import js from '@eslint/js';

export default [
    js.configs.recommended,
    {
        languageOptions: {
            ecmaVersion: 2022,
            sourceType: 'module',
            globals: {
                window: 'readonly',
                document: 'readonly',
                console: 'readonly',
                fetch: 'readonly'
            }
        },
        rules: {
            'no-unused-vars': 'error',
            'no-console': 'warn',
            'prefer-const': 'error',
            'no-var': 'error',
            'semi': ['error', 'always'],
            'quotes': ['error', 'single'],
            'indent': ['error', 2],
            'max-len': ['error', { 'code': 100 }]
        }
    }
];
EOF

# Run ESLint
npx eslint . --fix
```

#### 3.2 Prettier Configuration
```bash
# Install Prettier
npm install --save-dev prettier

# Create .prettierrc
cat > .prettierrc << 'EOF'
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false
}
EOF

# Create .prettierignore
cat > .prettierignore << 'EOF'
node_modules
dist
build
coverage
*.min.js
EOF

# Format code
npx prettier --write .
```

## 🧪 Testing Frameworks and Strategies

### Python Testing with pytest

#### 1.1 Unit Tests
```python
# tests/unit/test_app.py
import pytest
import json
from src.backend.app import app

@pytest.fixture
def client():
    """Create test client"""
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

class TestAPI:
    """Unit tests for API endpoints"""
    
    def test_status_endpoint(self, client):
        """Test status endpoint returns correct response"""
        response = client.get('/api/status')
        assert response.status_code == 200
        
        data = json.loads(response.data)
        assert data['status'] == 'healthy'
        assert 'timestamp' in data
        assert 'version' in data
    
    def test_users_endpoint(self, client):
        """Test users endpoint returns user list"""
        response = client.get('/api/users')
        assert response.status_code == 200
        
        data = json.loads(response.data)
        assert 'users' in data
        assert len(data['users']) > 0
        assert data['users'][0]['name'] == 'Alice'
    
    def test_nonexistent_endpoint(self, client):
        """Test 404 for nonexistent endpoint"""
        response = client.get('/api/nonexistent')
        assert response.status_code == 404

class TestUtilityFunctions:
    """Unit tests for utility functions"""
    
    def test_data_validation(self):
        """Test data validation logic"""
        # Add utility function tests here
        pass
    
    @pytest.mark.parametrize("input_data,expected", [
        ("valid@email.com", True),
        ("invalid-email", False),
        ("", False),
    ])
    def test_email_validation(self, input_data, expected):
        """Test email validation with different inputs"""
        # Implement email validation test
        pass
```

#### 1.2 Integration Tests
```python
# tests/integration/test_database.py
import pytest
import tempfile
import os
from src.backend.app import app

@pytest.fixture
def app_with_db():
    """Create app with test database"""
    db_fd, app.config['DATABASE'] = tempfile.mkstemp()
    app.config['TESTING'] = True
    
    with app.app_context():
        # Initialize test database
        pass
    
    yield app
    
    os.close(db_fd)
    os.unlink(app.config['DATABASE'])

class TestDatabaseIntegration:
    """Integration tests with database"""
    
    def test_user_creation_and_retrieval(self, app_with_db):
        """Test creating and retrieving users from database"""
        with app_with_db.test_client() as client:
            # Test user creation
            response = client.post('/api/users', json={
                'name': 'Test User',
                'email': 'test@example.com'
            })
            assert response.status_code == 201
            
            # Test user retrieval
            response = client.get('/api/users')
            assert response.status_code == 200
            # Add more assertions
```

### JavaScript Testing with Jest

#### 2.1 Jest Configuration
```bash
# Install Jest
npm install --save-dev jest @jest/globals

# Create jest.config.js
cat > jest.config.js << 'EOF'
export default {
  testEnvironment: 'jsdom',
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  testMatch: [
    '**/__tests__/**/*.js',
    '**/?(*.)+(spec|test).js'
  ],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js']
};
EOF

# Create test setup
mkdir -p tests
cat > tests/setup.js << 'EOF'
// Global test setup
global.fetch = jest.fn();

// Mock console methods in tests
global.console = {
  ...console,
  log: jest.fn(),
  error: jest.fn(),
  warn: jest.fn(),
};
EOF
```

#### 2.2 Frontend Unit Tests
```javascript
// tests/frontend.test.js
import { jest } from '@jest/globals';

describe('Frontend Application', () => {
  beforeEach(() => {
    // Reset DOM
    document.body.innerHTML = '';
    
    // Reset fetch mock
    fetch.mockClear();
  });

  test('should load API status successfully', async () => {
    // Mock successful API response
    fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => ({
        status: 'healthy',
        timestamp: '2024-01-01T00:00:00Z'
      })
    });

    // Load the HTML
    document.body.innerHTML = `
      <div id="api-status">Loading API status...</div>
    `;

    // Simulate the fetch call (you'd need to extract this to a function)
    const response = await fetch('/api/status');
    const data = await response.json();
    
    document.getElementById('api-status').innerHTML = 
      `<p>API Status: <strong>${data.status}</strong></p>`;

    expect(document.getElementById('api-status').innerHTML)
      .toContain('API Status: <strong>healthy</strong>');
  });

  test('should handle API error gracefully', async () => {
    // Mock API error
    fetch.mockRejectedValueOnce(new Error('Network error'));

    document.body.innerHTML = `
      <div id="api-status">Loading API status...</div>
    `;

    try {
      await fetch('/api/status');
    } catch (error) {
      document.getElementById('api-status').innerHTML = 
        '<p>API Status: <strong>Offline</strong></p>';
    }

    expect(document.getElementById('api-status').innerHTML)
      .toContain('API Status: <strong>Offline</strong>');
  });
});
```

## 📊 Code Coverage and Quality Gates

### Coverage Configuration
```bash
# Python coverage configuration (.coveragerc)
cat > .coveragerc << 'EOF'
[run]
source = src
omit = 
    */tests/*
    */venv/*
    */env/*
    setup.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError

[html]
directory = htmlcov
EOF

# Run coverage
pytest --cov=src --cov-report=html --cov-report=xml
```

### Quality Gates Script
```bash
#!/bin/bash
# quality-gate.sh

set -e

echo "🔍 Running Quality Gates..."

# Variables
MIN_COVERAGE=80
MAX_COMPLEXITY=10
SONAR_TOKEN=${SONAR_TOKEN:-"your-token"}

# Run tests and generate coverage
echo "📊 Running tests and generating coverage..."
pytest --cov=src --cov-report=xml --cov-report=term-missing

# Check coverage threshold
COVERAGE=$(python -c "
import xml.etree.ElementTree as ET
tree = ET.parse('coverage.xml')
root = tree.getroot()
coverage = float(root.attrib['line-rate']) * 100
print(f'{coverage:.1f}')
")

echo "Coverage: ${COVERAGE}%"
if (( $(echo "$COVERAGE < $MIN_COVERAGE" | bc -l) )); then
    echo "❌ Coverage ${COVERAGE}% is below minimum ${MIN_COVERAGE}%"
    exit 1
else
    echo "✅ Coverage ${COVERAGE}% meets minimum requirement"
fi

# Run static analysis
echo "🔍 Running static analysis..."
pylint src/backend/ --exit-zero --output-format=json > pylint-report.json

# Check complexity
echo "📈 Checking code complexity..."
radon cc src/backend/ --min B --json > complexity-report.json

# Run SonarQube analysis (if configured)
if [ ! -z "$SONAR_TOKEN" ]; then
    echo "🎯 Running SonarQube analysis..."
    sonar-scanner \
        -Dsonar.projectKey=devops-demo-project \
        -Dsonar.sources=src \
        -Dsonar.host.url=http://localhost:9000 \
        -Dsonar.login=$SONAR_TOKEN
fi

echo "✅ All quality gates passed!"
```

## 🔄 Automated Quality Checks

### Pre-commit Hooks
```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict

  - repo: https://github.com/psf/black
    rev: 23.7.0
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/pycqa/pylint
    rev: v3.0.0a7
    hooks:
      - id: pylint
        args: [--errors-only]

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.44.0
    hooks:
      - id: eslint
        files: \.(js|jsx)$
        additional_dependencies:
          - eslint@8.44.0
EOF

# Install hooks
pre-commit install
```

### GitHub Actions Quality Workflow
```yaml
# .github/workflows/quality.yml
name: Code Quality

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  quality:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r src/backend/requirements.txt
        pip install pytest pytest-cov pylint black
    
    - name: Run Black formatter check
      run: black --check src/backend/
    
    - name: Run Pylint
      run: pylint src/backend/ --exit-zero
    
    - name: Run tests with coverage
      run: |
        pytest --cov=src --cov-report=xml --cov-report=term-missing
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
    
    - name: SonarQube Scan
      uses: sonarqube-quality-gate-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is code coverage and why is it important?**
A: Code coverage measures the percentage of code executed during testing. It's important because it helps identify untested code, ensures comprehensive testing, and indicates code quality, though 100% coverage doesn't guarantee bug-free code.

**Q2: What is the difference between unit tests and integration tests?**
A: Unit tests test individual components in isolation, are fast and focused on single functions/methods. Integration tests test how multiple components work together, are slower but catch interface and interaction issues.

**Q3: What is static code analysis?**
A: Static code analysis examines code without executing it to find potential bugs, security vulnerabilities, code smells, and style violations. Tools like SonarQube, Pylint, and ESLint perform static analysis.

**Q4: What are code smells?**
A: Code smells are indicators of poor code quality that don't prevent functionality but make code harder to maintain. Examples include long methods, duplicate code, large classes, and complex conditionals.

**Q5: What is technical debt?**
A: Technical debt represents the cost of additional work caused by choosing quick solutions over better approaches. It accumulates over time and requires refactoring to maintain code quality and development velocity.

### Intermediate Level (6-15)

**Q6: How do you implement quality gates in CI/CD pipelines?**
A: Set minimum thresholds for coverage, complexity, and security issues. Use tools like SonarQube quality gates, fail builds that don't meet criteria, implement automated testing at multiple levels, and require code reviews.

**Q7: What is test-driven development (TDD)?**
A: TDD is a development approach where you write tests before code: Red (write failing test), Green (write minimal code to pass), Refactor (improve code while keeping tests passing). Benefits include better design and comprehensive test coverage.

**Q8: How do you measure code quality?**
A: Use metrics like code coverage, cyclomatic complexity, duplication percentage, maintainability index, technical debt ratio, and defect density. Combine quantitative metrics with qualitative code reviews.

**Q9: What are the best practices for writing maintainable code?**
A: Follow SOLID principles, write clear and descriptive names, keep functions small and focused, minimize dependencies, write comprehensive tests, document complex logic, and refactor regularly.

**Q10: How do you handle flaky tests?**
A: Identify root causes (timing issues, external dependencies, test isolation), fix underlying issues, implement proper waits and retries, use test isolation techniques, and quarantine flaky tests while fixing them.

## 🔑 Key Takeaways

- **Quality First**: Implement quality checks early in development process
- **Automation**: Automate testing and quality analysis in CI/CD pipelines
- **Comprehensive Testing**: Combine unit, integration, and end-to-end tests
- **Continuous Monitoring**: Track quality metrics over time
- **Team Culture**: Foster quality-focused development culture
- **Tool Integration**: Use appropriate tools for different quality aspects

## 🚀 Next Steps

- Day 24: Hands-on Lab - Git Workflow Implementation
- Day 25: Docker Fundamentals and Containerization
- Implement advanced testing strategies and quality metrics

---

**Hands-on Completed:** ✅ Code Quality Tools, Testing Frameworks, Quality Gates  
**Duration:** 4-5 hours  
**Difficulty:** Intermediate to Advanced