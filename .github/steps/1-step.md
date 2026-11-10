### From Vague Requirement to Implementation Plan

In real projects, requirements rarely arrive as clean specification documents. They come as Slack messages, emails, or casual conversations. A key skill is turning these vague inputs into structured, actionable plans — and AI can accelerate every step of this process.

In this exercise you will practice using AI to:

1. **Analyze impact** against existing specifications
2. **Generate clarifying questions** for the customer
3. **Produce an implementation plan** with phased, parallelizable work

## Context

The `docs/` directory contains the specification documents for a Dynamic Form Application:

- `RequirementsSpecification.md` — functional and non-functional requirements
- `TechnicalSpecification.md` — architecture, data model, and API details
- `UseCases.md` — actor-based use case descriptions

A customer has sent the following message:

> "Hey, we have some forms that should only be open for a limited time but right now there's no way to control that — people keep submitting to old forms. Would it be possible to add some kind of start/end date thing to forms?"

## Exercise

<!-- github-only -->
### Setup — clone your copy

Clone your copy of this repository (the one created from the template) to your computer and open it in a terminal:

```
git clone <URL of your copy>
cd <name of your repo>
```

You can copy the URL from the green **Code** button on your repository page.
<!-- /github-only -->
<!-- mcp-only
### Setup

Claude has already written the starter files into this folder. Open a terminal here.
-->

### Task

Using AI as your collaborator, work through the following steps. Create all output files in the `docs/` directory.

**Step 1 — Impact Analysis & Clarifying Questions**

Give the customer message and the existing specification documents to AI. Ask it to:

- Analyze which parts of the system would need to change (database, backend, frontend, API, tests, etc.)
- Identify ambiguities and edge cases in the request
- Generate a list of clarifying questions to send back to the customer

Review the AI's output. Does it cover areas you wouldn't have thought of? Did it miss anything obvious?

**Step 2 — Incorporate Customer Answers**

Now imagine the customer has responded to the clarifying questions from Step 1. Play the role of the customer — review the questions AI generated and come up with reasonable answers. Think about what would make sense for a typical organization using this form application.

Feed the answers back to AI and ask it to create an implementation plan in `docs/ImplementationPlan.md`. The plan should cover:

- Design decisions summary
- Database, backend, frontend, and API changes
- Testing plan
- Step-by-step implementation order

**Step 3 — Organize into Phases**

Ask AI to reorganize the implementation plan into phases, identifying which tasks can be executed concurrently. The final plan should show:

- Clear phase boundaries with entry/exit criteria
- Parallel work tracks within each phase
- Dependencies between phases
- A concurrency map or diagram showing the critical path

Save the final phased plan in `docs/ImplementationPlan.md`.

**Step 4 — Create a Task Tracking Checklist**

Ask AI to generate a task checklist file `docs/Tasks.md` based on the phased implementation plan. The file should:

- Be organized by phase, matching the phases in the implementation plan
- Use markdown checkboxes (`- [ ]`) for every individual task
- Include the parallel track grouping within each phase so developers can see which tasks are independent
- Be practical to use during implementation — a developer should be able to open this file, check off tasks as they complete them, and see at a glance what remains

This gives the team a ready-made way to track implementation progress. The tasks can be used as-is with the markdown checkboxes, or copied into your project management tool (Jira, Linear, Azure DevOps, etc.) as individual work items.

<!-- github-only -->
### Automated Checks

The CI pipeline will verify:

1. `docs/ImplementationPlan.md` exists and is non-empty
2. The plan contains at least 3 distinct phases
3. The plan references concurrency or parallel execution
4. The plan covers backend, frontend, and testing areas
5. `docs/Tasks.md` exists
6. `docs/Tasks.md` is organized by phases matching the implementation plan

**Note:** There is no single "correct" plan — the checks validate structure and completeness, not specific content. Your plan may differ from someone else's and still be equally valid.

<details>
<summary>Having trouble? 🤷</summary><br/>

 * Start by giving AI all three specification documents at once along with the customer message — it needs the full picture to do a thorough impact analysis.
 * If the clarifying questions seem too generic, prompt AI to focus on edge cases: "What happens if a user has the form open when it expires?" or "How should this interact with the existing admin views?"
 * When asking for phases, be explicit: "Which tasks have no dependencies on each other and can be done by separate developers at the same time?"
 * For the task checklist, try prompting: "Turn the phased plan into a markdown file with checkboxes grouped by phase and track, so we can use it to track progress during implementation."
 * If nothing happens for a while after committing your code, or if the tests don't pass, check the [Actions](../actions) tab to see what went wrong. 
 * If you are still having trouble, ask for help in the course support channel.

</details>

---

<img alt="Amin 2.0" src="../images/amin2_smile.png" align="right" height="125px" />

Please, follow the steps above.
I'll watch your progress in the background to provide feedback. 🧐
<!-- /github-only -->
<!-- mcp-only
When you are done, ask me to check your solution.
-->
