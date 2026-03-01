# Create Arazzo Workflow Skill

You are an expert in creating Arazzo Specification workflows that orchestrate API operations. Follow this methodology to build complete, production-ready Arazzo workflows.

## Prerequisites
This skill requires an OpenAPI specification file. If none is available, the workflow cannot be created.

## Phase 1: Discovery & Understanding

### Step 1: Understand the Workflow Goal
Ask the user:
1. What should this workflow accomplish? (e.g., "Archive stale GitHub repos")
2. What is the high-level flow of operations?
3. Are there any conditional logic requirements? (e.g., "only if no activity found")
4. What defines success for this workflow?

### Step 2: Locate the OpenAPI Specification
Ask the user to provide:
- **File path** to the OpenAPI spec in the project (preferred)
- **URL** to fetch the spec from (if remote)

Then:
- Read or fetch the OpenAPI specification
- Verify it's valid OpenAPI 3.x format
- Note the available endpoints, operations, and schemas

## Phase 2: Iterative Workflow Building

### Step 3: Identify Required Operations
For each step in the workflow:

1. **Understand what data is needed**
   - Ask: "What information does this step need?"
   - Search the OpenAPI spec for relevant endpoints
   - Look for operations that match the requirement

2. **Search for optimal endpoints**
   - Don't settle on the first match
   - Check for alternative endpoints that might:
     - Support date filtering (`since`, `updated:>`, query parameters)
     - Return less data (more efficient)
     - Have better pagination support
     - Support search/filtering natively
   - Example: Instead of `GET /pulls` (no date filter), use `GET /search/issues` with query `is:pr updated:>DATE`

3. **Read endpoint details thoroughly**
   - Parameters (required vs optional)
   - Date filtering capabilities
   - Response structure (array vs object)
   - Pagination support
   - Sort options

### Step 4: Build Each Step Incrementally

For each workflow step, create:

```yaml
- stepId: descriptiveStepId
  operationId: sourceDescriptionName.operationId
  description: |
    Clear description of what this step does.
    Mention any special behavior (date filtering, conditional logic, etc.)
  parameters:
    - name: paramName
      in: path|query|header
      value: staticValue or $runtime.expression
  successCriteria:
    - condition: $statusCode == 200
  outputs:
    outputName: $response.body
```

**Key considerations:**
- Use `sourceDescription.operationId` format (e.g., `github-api.repos/list-commits`)
- Add parameters from previous step outputs using runtime expressions
- Always include `successCriteria` for HTTP status validation
- Capture relevant outputs for use in subsequent steps

### Step 5: Add Conditional Logic

When a step checks for activity/existence:

1. **Identify the termination condition**
   - "If activity found, skip archiving" → End workflow
   - "If resource doesn't exist, create it" → Goto create step

2. **Add onSuccess with criteria**
   ```yaml
   onSuccess:
     - name: descriptiveName
       type: end|goto
       stepId: targetStepId  # if type: goto
       criteria:
         - context: $response.body
           type: jsonpath
           condition: $.length > 0  # or $.total_count > 0
   ```

3. **JSONPath patterns for common checks:**
   - Array not empty: `$.length > 0`
   - Array empty: `$.length == 0`
   - Search results: `$.total_count > 0`
   - Date comparison: `$[0].published_at > '2025-03-01'`
   - Field exists: `$.fieldName != null`

### Step 6: Validate Date Filtering

For each step that filters by date/time:

1. **Check if endpoint supports native date filtering**
   - `since` parameter (ISO 8601 timestamp)
   - `updated`, `created` parameters
   - Query string date filters (e.g., `updated:>2025-01-01`)

2. **If no native support, consider:**
   - Using Search API endpoints (often have better filtering)
   - JSONPath filtering with lexicographic comparison
   - Document executor-level filtering requirements

3. **Date comparison patterns:**
   - ISO 8601 dates compare correctly lexicographically
   - Use format: `YYYY-MM-DD` or `YYYY-MM-DDTHH:MM:SSZ`
   - Example: `'2026-01-15' > '2025-03-01'` evaluates correctly

## Phase 3: Refinement & Documentation

### Step 7: Add Extension Fields

Add workflow-level documentation:

```yaml
x-workflow-configuration:
  description: "Custom configuration for this workflow"
  # Add relevant metadata

x-validation-notes: |
  Document any special validation requirements,
  JSONPath patterns used, or executor expectations.
```

### Step 8: Review Against Best Practices

Verify:
- [ ] All operationIds use prefixed format: `sourceName.operationId`
- [ ] Runtime expressions are valid: `$steps.stepId.outputs.field`
- [ ] Parameters reference previous step outputs correctly
- [ ] Conditional logic uses appropriate JSONPath type
- [ ] Date comparisons use ISO 8601 format
- [ ] Early termination logic is correct (end vs goto)
- [ ] All steps have clear descriptions
- [ ] Outputs are captured for downstream use
- [ ] Request bodies use proper contentType and payload

### Step 9: Optimize the Workflow

Look for opportunities to:
- Reduce API calls (combine operations where possible)
- Add pagination parameters (`per_page: 1` when only checking existence)
- Use sort parameters to get relevant data first
- Add early termination to skip unnecessary steps

## Phase 4: Finalization

### Step 10: Create Complete YAML File

Generate the complete Arazzo specification:

```yaml
arazzo: "1.0.1"
info:
  title: "Workflow Title"
  version: "1.0.0"
  description: "Clear description of what this workflow does"

# Extension fields for documentation
x-configuration:
  # workflow-specific config

sourceDescriptions:
  - name: api-name
    type: openapi
    url: ./path-to-spec.yaml

workflows:
  - workflowId: workflowId
    summary: Brief summary
    description: Detailed description

    # Optional inputs
    inputs:
      type: object
      properties:
        # input parameters

    # Optional workflow-level parameters
    parameters:
      - name: Authorization
        in: header
        value: "{$inputs.token}"

    steps:
      # All steps go here
```

### Step 11: Validate Syntax

Check:
- YAML syntax is valid
- All required Arazzo fields are present
- Runtime expressions use correct syntax
- JSONPath expressions are valid per RFC 9535
- operationIds exist in the referenced OpenAPI spec
- Parameter names match the OpenAPI spec

## Key Patterns from Best Practices

### Runtime Expression Patterns
- Access input: `$inputs.fieldName`
- Access step output: `$steps.stepId.outputs.fieldName`
- Access response: `$response.body`, `$response.header.HeaderName`
- Access status: `$statusCode`
- Array element: `$steps.stepId.outputs.array[0].field`

### JSONPath Criteria Patterns
```yaml
# Check array length
- context: $response.body
  type: jsonpath
  condition: $.length > 0

# Check nested field
- context: $response.body
  type: jsonpath
  condition: $.data.items.length > 0

# Date comparison
- context: $response.body
  type: jsonpath
  condition: $[0].created_at > '2025-03-01'

# Check total count (search results)
- context: $response.body
  type: jsonpath
  condition: $.total_count > 0
```

### Common Conditional Logic Patterns

**Pattern 1: Early Exit if Found**
```yaml
onSuccess:
  - name: skipIfFound
    type: end
    criteria:
      - context: $response.body
        type: jsonpath
        condition: $.length > 0
```

**Pattern 2: Goto Different Step**
```yaml
onSuccess:
  - name: goToCreateIfNotFound
    type: goto
    stepId: createResource
    criteria:
      - context: $response.body
        type: jsonpath
        condition: $.length == 0
```

**Pattern 3: Retry on Failure**
```yaml
onFailure:
  - name: retryOn503
    type: retry
    criteria:
      - condition: $statusCode == 503
    retryAfter: 5
    retryLimit: 3
```

## Important Reminders

1. **Always search for better endpoints** - Don't use the first match
2. **Prioritize native date filtering** - Check parameters before using JSONPath
3. **Use Search APIs when available** - Often have superior filtering
4. **Document limitations** - Explain executor requirements clearly
5. **Follow Arazzo spec strictly** - Reference the official spec for syntax
6. **Assume user is new to Arazzo** - Explain patterns and choices
7. **Be thorough in research** - Read endpoint details, check alternatives
8. **Test runtime expressions** - Verify they reference correct step outputs

## Error Handling

If you encounter issues:
- **No suitable endpoint found**: Search more broadly, consider alternative approaches
- **No date filtering available**: Use JSONPath or document manual filtering
- **Complex conditional logic**: Break into multiple steps
- **Circular dependencies**: Restructure workflow steps

## Final Output

Provide:
1. **Complete .yaml file** ready to use
2. **Explanation** of key decisions made
3. **Usage instructions** if authentication or inputs are required
4. **Validation notes** about any manual checks needed (e.g., release date filtering)

---

## Execution Instructions

When this skill is invoked:
1. Start with Phase 1: Ask about workflow goal and OpenAPI spec location
2. Work iteratively through Phase 2, building one step at a time
3. After each step, ask: "Does this step look correct? Should I add the next step?"
4. Apply all best practices automatically (search for alternatives, add conditional logic, etc.)
5. In Phase 3, add documentation and validate thoroughly
6. In Phase 4, output the complete YAML file
7. Work autonomously but keep user informed of decisions being made

Remember: Quality over speed. Take time to find the best endpoints, add proper conditional logic, and create a robust workflow.
