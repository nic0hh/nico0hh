# Job Posting Fraud Signals

A small SQL analysis in a Jupyter notebook, asking which features of a job posting are associated with fraud.

**Data:** [Real or Fake Job Posting dataset](https://www.kaggle.com/datasets/shivamb/real-or-fake-fake-jobposting-prediction) (Kaggle): about 18,000 postings from one recruitment platform, 866 of them labelled fraudulent. The data file is not included in this repository.

**Method:** the CSV is loaded with pandas into an in-memory SQLite database, and each question is answered with a SQL query followed by a short interpretation.

## Findings

- A missing company logo is the strongest signal tested: 15.9% of postings without a logo are fraudulent, compared with 2.0% of postings with one (4.8% across all postings).
- An unspecified industry is an unreliable indicator on its own (5.6%), and adds little when combined with a missing logo (16.4%).

## Limitations

- The data comes from a single platform and is over a decade old; scam patterns have changed since.
- The labels reflect what that platform caught, so some fraud may be labelled as legitimate.
- These signals indicate risk and could prioritise postings for review, but they don't prove fraud on their own.
