# Automated Testing

## Running tests locally

Install test dependencies:
```bash
pip install . --group test
```

Run inference tests:
```bash
task inference
```

## Quality regression test
Compares images in 2 directories to ensure they are the same

1) Run an inference test to save a directory of "ground truth" images
```bash
task inference -- --output_dir tests/inference/baseline
```
2) Make code edits

3) Run inference and quality comparison tests
```bash
task quality
```
