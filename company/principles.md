# Principles

1. **Measured evidence beats intuition.**

2. **Every performance claim requires a valid comparison.**

   * baseline
   * candidate
   * equivalent workload
   * controlled environment
   * reproducible execution

3. **Performance results are experimental evidence, not absolute truth.**
   Account for variance, measurement error, and statistical significance.

4. **AI hypotheses are not facts.**
   AI-generated explanations must be supported by observable evidence.

5. **Experiments are immutable.**
   Record the code, engine version, configuration, workload, hardware,
   environment, results, and analysis required to reproduce an experiment.

6. **Optimize performance and cost together.**
   Faster is not necessarily better if it requires disproportionately
   more infrastructure.

7. **Deterministic execution, intelligent investigation.**
   Agents may decide what experiment to run, but the execution platform
   must execute experiments deterministically.

8. **Human approval is required for:**

   * destructive cloud actions
   * material spending
   * production changes
   * releases

9. **Knowledge belongs to the company.**
   Important findings, decisions, failures, and solutions become durable
   organizational knowledge.

10. **Repeated lessons become playbooks.**
    The company should become more capable through every experiment.

11. **Disprove before declaring success.**
    Performance claims should actively test alternative explanations and
    attempt to reproduce unexpected results.

