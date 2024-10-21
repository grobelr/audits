# Audit Findings Repository

This repository contains a collection of audit findings from various smart contract security reviews I've conducted. The findings are organized by project and severity level, providing insights into common vulnerabilities and best practices in smart contract development.

## Important Note

These findings were submitted to platforms such as Code4rena, Sherlock, Immunefi, and others. It's important to note that submission does not guarantee acceptance. Some findings may have been rejected or classified differently by the respective platforms or project teams.

## Repository Structure

The repository is organized as follows:

- Each project has its own directory named after the project (e.g., `2023-10-zksync`, `2023-09-maia`)
- Within each project directory, findings are categorized into separate files based on severity:
  - `MH.md`: Medium and High severity findings
  - `QA-report.md`: Quality Assurance and Low severity findings
  - `GAS.md`: Gas optimization suggestions

## Types of Findings

1. **Security Vulnerabilities**: Critical and high-risk issues that could lead to loss of funds or compromise of contract functionality.
2. **Code Quality Issues**: Recommendations for improving code readability, maintainability, and adherence to best practices.
3. **Gas Optimizations**: Suggestions for reducing gas costs and improving contract efficiency.

## How to Use This Repository

- Browse through different project directories to see findings from specific audits.
- Use the severity-based files (MH.md, QA-report.md, GAS.md) to focus on particular types of issues.
- Each finding typically includes:
  - A description of the issue
  - Its potential impact
  - Code snippets or references to affected lines
  - Recommended fixes or mitigations

## Contributing

While this repository primarily contains my personal audit findings, contributions or discussions are welcome. If you have insights or alternative perspectives on any finding, feel free to open an issue or submit a pull request.

## Disclaimer

These audit findings are provided for educational and informational purposes only. They reflect the state of the audited contracts at the time of review and may not be applicable to updated or modified versions of the smart contracts. The acceptance status of each finding by the respective audit platforms or project teams may vary.