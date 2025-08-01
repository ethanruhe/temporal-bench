# temporal-bench

An automated benchmark for evaluating how effectively foundational LLMs can integrate Temporal into existing codebases.

## Benchmark Score

The benchmark generates two types of scores for each model:

- **Language Score**: For each programming language, individual test case scores (0-2) are summed and normalized to a 0-100 scale
- **Aggregate Score**: The total score across all languages, also normalized to 0-100

**Important**: Scores are a function of the specific test set used. As the test set expands, the benchmark will be versioned accordingly. Scores across different benchmark versions are not comparable since the underlying tests have changed. However, within any given benchmark version, scores provide a reliable basis for comparing performance across different models.


## Setup
### Python Environment
This project uses a Python virtual environment to manage dependencies. Follow these steps to set up your development environment:

#### Create and activate the virtual environment
```bash
# Create a virtual environment named 'temporal-bench'
python -m venv temporal-bench

# Activate the virtual environment
# On macOS/Linux:
source temporal-bench/bin/activate
```

#### Deactivate the virtual environment
When you're done working on the project, deactivate the virtual environment:
```bash
deactivate
```

### Python Dependencies
#### Creating requirements.txt
To capture your project's dependencies in a requirements file:
```bash
# After installing packages with pip, generate requirements.txt
pip freeze > requirements.txt
```

#### Installing from requirements.txt
To install all required packages from the requirements file:
```bash
# Make sure your virtual environment is activated first
pip install -r requirements.txt
```

#### Adding new dependencies
When adding new Python packages:
```bash
# Install the package
pip install package-name

# Update requirements.txt
pip freeze > requirements.txt
```

## Project Documentation
For detailed information about the benchmark strategy, architecture, and implementation, see the [Product Requirements Document](prd.md).

## Quick Start
1. Clone the repository
2. Create and activate the Python virtual environment (see above)
3. Install dependencies: `pip install -r requirements.txt`
4. Set up your `.env` file with API keys for the LLM services
5. Run the benchmark: `python main.py`

Results will be generated in the `results/` directory.