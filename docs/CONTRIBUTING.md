# Contributing to UniClaw 🦞

Thank you for your interest in contributing to UniClaw! This skill was built by the community, for the community. We welcome contributions of all kinds.

## 🎯 How Can I Contribute?

### 1. Report Bugs or Issues

Found a calculation error or unexpected behavior?

**Before submitting:**
- Check if the issue already exists in [GitHub Issues](https://github.com/your-org/uniclaw-skill/issues)
- Test with the latest version of the skill
- Gather relevant information (position data, error messages, Claude version)

**Submit a good bug report:**
```markdown
**Issue:** Fee calculation incorrect when position out of range

**Steps to Reproduce:**
1. Ask UniClaw to analyze position with current_tick < tickLower
2. Provide feeGrowthGlobal and position data
3. Observe incorrect fee calculation

**Expected Behavior:**
Fees should be 0 when position is out of range

**Actual Behavior:**
Returns non-zero fee value

**Environment:**
- Claude Version: Desktop 2.x
- Skill Version: 1.0.0
- Position Data: [paste here]
```

### 2. Suggest New Features

Have an idea for new risk metrics or strategies?

**Feature Request Template:**
```markdown
**Feature:** Add Sortino Ratio to performance metrics

**Use Case:** 
Sortino ratio only penalizes downside volatility, making it 
better than Sharpe for LP positions with asymmetric returns.

**Implementation Ideas:**
- Add sortino_ratio() method to PositionMetrics class
- Use only negative returns in std deviation calculation
- Include in standard performance report

**Resources:**
- [Link to academic paper on Sortino]
- [Example calculation]
```

### 3. Improve Documentation

Documentation is crucial for skill adoption!

**Areas needing improvement:**
- Add more real-world examples
- Create video walkthroughs
- Translate to other languages
- Write blog posts about using UniClaw
- Improve code comments in SKILL.md

**How to contribute docs:**
1. Fork the repository
2. Create a new branch: `docs/improve-risk-framework-explanation`
3. Make your changes
4. Submit a Pull Request

### 4. Add New Risk Metrics

UniClaw's risk framework can always be enhanced!

**Popular requests:**
- Omega ratio
- Calmar ratio
- Downside deviation
- Maximum adverse excursion
- Correlation analysis (ETH/USDC correlation with volatility)

**Guidelines:**
- Include mathematical formula in comments
- Provide academic reference or source
- Add to `calculate_all_metrics()` method
- Create example output
- Update eval expectations if needed

### 5. Improve Backtesting

Help make backtesting more realistic!

**Ideas:**
- Add slippage modeling
- Include gas cost variations
- Model MEV impact
- Support multiple positions simultaneously
- Add realistic order execution delays

### 6. Create New Evals

More test cases = better skill quality!

**Good eval characteristics:**
- Tests specific skill capability
- Clear expected outputs
- Representative of real-world usage
- Covers edge cases

**Eval template:**
```json
{
  "id": 5,
  "prompt": "Clear, specific user query here",
  "expectations": [
    "Specific testable expectation 1",
    "Specific testable expectation 2",
    "Mathematical calculation must be present",
    "Recommendation must be YES/NO with reasoning"
  ]
}
```

---

## 🔧 Development Workflow

### Setting Up Your Environment

```bash
# 1. Fork the repository
# Click "Fork" button on GitHub

# 2. Clone your fork
git clone https://github.com/YOUR-USERNAME/uniclaw-skill.git
cd uniclaw-skill

# 3. Add upstream remote
git remote add upstream https://github.com/orki-finance/uniclaw-skill.git

# 4. Create a branch
git checkout -b feature/add-sortino-ratio
```

### Making Changes

1. **Edit SKILL.md** - This is the main skill file
2. **Test thoroughly** - Ask Claude various questions using your modified skill
3. **Update evals** - If you changed capabilities, update eval expectations
4. **Document** - Add examples and explain new features

### Code Style Guidelines

**For Python code blocks in SKILL.md:**
```python
# Good: Clear variable names, commented formulas
def calculate_sortino_ratio(returns, target_return=0.0):
    """
    Sortino Ratio = (Return - Target) / Downside Deviation
    
    Only penalizes downside volatility, not upside.
    """
    import numpy as np
    
    # Filter only negative returns (downside)
    downside_returns = returns[returns < target_return]
    
    # Calculate downside deviation
    downside_std = np.std(downside_returns)
    
    # Sortino formula
    mean_return = np.mean(returns)
    sortino = (mean_return - target_return) / downside_std
    
    return sortino
```

**For documentation:**
- Use clear, concise language
- Include mathematical formulas where relevant
- Provide real-world examples
- Link to academic sources for complex concepts

### Testing Your Changes

**Manual testing:**
```bash
# 1. Copy your modified SKILL.md to Claude's skill folder
cp SKILL.md ~/Library/Application\ Support/Claude/skills/uniclaw.md

# 2. Open Claude and test
# Ask questions that exercise your new feature

# 3. Verify outputs match expectations
```

**Automated testing (if available):**
```bash
# Run eval suite
python test_skill.py --eval all

# Run specific eval
python test_skill.py --eval 1
```

### Submitting a Pull Request

```bash
# 1. Commit your changes
git add SKILL.md
git commit -m "Add Sortino ratio to performance metrics"

# 2. Push to your fork
git push origin feature/add-sortino-ratio

# 3. Create Pull Request on GitHub
# - Clear title describing the change
# - Reference any related issues
# - Include example outputs
# - Explain why the change is useful
```

**Good PR Description:**
```markdown
## Summary
Adds Sortino ratio to performance metrics for better downside risk assessment.

## Motivation
Sharpe ratio penalizes both upside and downside volatility equally. 
For LP positions, we care more about downside risk. Sortino ratio 
only penalizes negative returns, making it more appropriate.

## Changes
- Added `sortino_ratio()` method to PositionMetrics class
- Updated `calculate_all_metrics()` to include Sortino
- Added example output to risk-assessment-example.md
- Updated eval #1 expectations

## Testing
Tested with 3 different positions:
1. High volatility position (42% vol) - Sortino: 1.2
2. Low volatility position (15% vol) - Sortino: 2.8
3. Position with drawdown - Sortino: 0.6

## Screenshots
[Paste example output]

Closes #42
```

---

## 🎨 Contribution Ideas by Skill Level

### Beginner
- Fix typos in documentation
- Add more examples to README
- Improve code comments
- Test skill on different position types
- Report bugs with detailed information

### Intermediate
- Add new eval cases
- Improve existing calculations (edge cases)
- Create visualization examples
- Write tutorial blog posts
- Translate documentation

### Advanced
- Implement new risk metrics (Omega, Calmar, etc.)
- Optimize mathematical calculations
- Add machine learning components
- Create automated testing framework
- Build companion tools (Discord bot, dashboard)

---

## 📋 Checklist Before Submitting

- [ ] Code follows the style guidelines
- [ ] Comments explain complex formulas
- [ ] Examples demonstrate the new feature
- [ ] Documentation updated (README, examples)
- [ ] Evals updated if capabilities changed
- [ ] Tested manually with Claude
- [ ] No breaking changes (or clearly documented)
- [ ] Academic references included for new metrics
- [ ] PR description is clear and detailed

---

## 🤝 Community Guidelines

### Be Respectful
- Treat all contributors with respect
- Welcome newcomers warmly
- Provide constructive feedback
- Assume good intentions

### Be Collaborative
- Share knowledge openly
- Help others learn
- Review PRs thoughtfully
- Celebrate contributions

### Be Professional
- Stay on topic
- Avoid personal attacks
- Focus on ideas, not people
- Keep discussions constructive

---

## 💬 Communication Channels

- **GitHub Issues:** Bug reports and feature requests
- **GitHub Discussions:** General questions and ideas
- **Pull Requests:** Code contributions
- **Discord:** (if available) Real-time chat
- **Twitter:** (if available) Announcements

---

## 🏆 Recognition

Contributors will be:
- Listed in CONTRIBUTORS.md
- Mentioned in release notes
- Credited in documentation
- Invited to maintainer team (for significant contributions)

---

## 📚 Resources for Contributors

### Learn About Uniswap
- [Uniswap V3 Whitepaper](https://uniswap.org/whitepaper-v3.pdf)
- [RareSkills Tutorials](https://rareskills.io/blog/uniswap-v3)
- [Official Docs](https://docs.uniswap.org)

### Learn About Risk Metrics
- *Options, Futures, and Other Derivatives* by John Hull
- *Quantitative Risk Management* by McNeil, Frey, Embrechts
- [Portfolio Visualizer](https://www.portfoliovisualizer.com) (for examples)

### Learn About Claude Skills
- [Anthropic Skills Documentation](https://docs.anthropic.com/claude/docs/skills)
- [Skill Creator Tool](https://github.com/anthropics/skill-creator)

---

## ❓ Questions?

Don't hesitate to ask! Open a [Discussion](https://github.com/your-org/uniclaw-skill/discussions) or reach out in Discord.

**Thank you for contributing to UniClaw! 🦞**

Every contribution, no matter how small, makes the DeFi community stronger.
