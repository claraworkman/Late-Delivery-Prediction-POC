# Contributing to Delivery Time Prediction POC

Thank you for your interest in contributing to this project!

## Getting Started

This is a Proof of Concept (POC) repository demonstrating ML workflows in Microsoft Fabric. While contributions are welcome, please note this is designed as a template/example project.

## How to Contribute

### Reporting Issues

If you find bugs or have suggestions:

1. Check if the issue already exists in GitHub Issues
2. Create a new issue with:
   - Clear title and description
   - Steps to reproduce (for bugs)
   - Expected vs actual behavior
   - Your Fabric environment details

### Suggesting Enhancements

For feature requests or improvements:

1. Open a GitHub Issue with the "enhancement" label
2. Describe the use case and benefit
3. Provide examples if applicable

### Code Contributions

If you'd like to contribute code:

1. **Fork the repository**
2. **Create a feature branch** (`git checkout -b feature/your-feature-name`)
3. **Make your changes**
   - Follow existing code style
   - Add comments for complex logic
   - Update documentation if needed
4. **Test your changes** in a Fabric workspace
5. **Commit your changes** (`git commit -m 'Add some feature'`)
6. **Push to your fork** (`git push origin feature/your-feature-name`)
7. **Open a Pull Request**

## Development Guidelines

### Code Style

- Follow PEP 8 for Python code
- Use descriptive variable names
- Add docstrings to functions
- Keep functions focused and modular

### Notebooks

- Clear markdown explanations between code cells
- Include expected outputs in comments
- Test with Microsoft Fabric runtime

### Documentation

- Update README.md for major changes
- Keep feature_list.json current
- Update model_card.md if model changes

## Testing

Before submitting:

- Test notebooks run end-to-end in Fabric
- Verify predictions write to Lakehouse correctly
- Check Power BI reports refresh successfully
- Validate MLflow model registration works

## Questions?

Open a GitHub Issue with the "question" label.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
