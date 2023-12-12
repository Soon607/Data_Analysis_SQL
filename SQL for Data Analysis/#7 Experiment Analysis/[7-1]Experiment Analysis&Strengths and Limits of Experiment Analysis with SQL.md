## Experimentation(A/B testing or split testing)
We can call it **The gold standard for establishing causality**
### Correlation vs Causation
Correlation: The statistical relationship between two variables that those variables are associated(Action A is related to Action B)
Causation: One thing causes another one.
**Correlation does not imply causation**

## Elements of the experiment analysis
### 1. Hypothesis
* A guess about behavioural change that will result from some alteration to a product, process, or message
* If the organization built it or has control over it, it can be experimented on, at least in theory.
### 2. A Success metric
The behavioural change we hypothesize might be related to form completion, purchase conversion, click-through, retention, engagement, or any other important behaviour to the organization's mission.    
<br/>
**The success metric should quantify this behaviour, be reasonably easy to measure, and be sensitive to detect a change**    
<br/>
Click-through, checkout completion, and time to complete a process are often good success metrics.  
However, retention and customer satisfaction are often less suitable success metrics, despite being very crucial, because they are frequently influenced by many factors beyond what is being tested in any individual experiment and thus are less sensitive to the changes we'd like to test    
<br/>
**Good success metrics are often ones that you already track as part of understanding company or organizational health**
### 3. A cohorting system
Establishing a system that randomly assigns entities to a control or experiment group and alters the experience accordingly.    
<br/>
To perform experiment analysis with SQL, the entity-level assignment data must flow into a table in the database that also contains behavioural data.

# Strengths and Limits of Experiment Analysis with SQL
In many cases with experiment analysis, the experiment cohort data and behavioural data are already flowing into a database, making SQL a natural choice    
<br/>
Success metrics are often already part of an organization's reporting and analysis vocabulary, with SQL queries already developed.    
<br/>
SQL is a good choice for automating experiment result reporting    
The same query can be run for each experiment, substituting the name or identifier of the experiment in the `WHERE` clauses.    
<br/>
However, SQL is not able to calculate statistical significance.
