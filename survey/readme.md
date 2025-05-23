# Form System: Developer Notes & To-Do List

This document outlines the basic workings of the "Tab-Based Form Builder" and the "Live Form Renderer" prototypes, followed by a to-do list for future enhancements and data considerations.

## I. System Overview

The system consists of two main components:

1. **Tab-Based Form Builder** (`tabbed_form_builder_v1`): An HTML/CSS/JS application that allows a user (e.g., an administrator or form designer) to visually create a multi-section questionnaire. It outputs a JSON definition of the form.

2. **Live Form Renderer** (`live_form_renderer_v1`): An HTML/CSS/JS application designed to take the JSON output from the builder and render it as an interactive, step-by-step form for the end-user (e.g., a parent seeking support).

## 1. Tab-Based Form Builder - How it Works

### Core Data Structure (`formSections`)

- The builder internally manages an array called `formSections`
- Each object in this array represents a "Section" (visually presented as a tab in the builder UI)
- A section object has:
  - `id` (e.g., `section_1`)
  - `name` (the title shown on the tab)
  - `elements` array

The `elements` array within each section holds objects representing individual questions or info blocks.

### Question/Element Structure

Each question/element object has:

- **`id`**: A unique identifier (e.g., `q_1`)
- **`label`**: The main text of the question or info block
- **`type`**: (e.g., `text`, `textarea`, `radio`, `checkboxes`, `select`, `info_block`, `section_header`)
- **`options`**: An array for types like `radio`, `checkboxes`, `select`. Each option object has:
  - `label`: Display text for the option
  - `value`: The data value of the option
  - `branch_to_element_id`: The ID of the section or question to jump to if this option is selected
- **`branch_to_element_id`**: For non-option-based elements (like `text`, `textarea`, `info_block`), this directly specifies the next element ID to go to after this element is processed
- **`required`**: A boolean (`true`/`false`) indicating if the question must be answered
- **`sectionId`**: Stores the ID of the section it belongs to
- **`is_start_of_phase`**: (Informational) Can be used to group questions logically or for progress indicators in the final form

### UI and Interaction

- **Sections (Tabs)**: Users can add, rename, and delete sections (tabs). The active section's questions are displayed for editing.
- **Questions**: Within each section, users can add questions, set their type, label, ID (auto-generated, read-only), and options.
- **Branching**:
  - For option-based questions, each option has a dropdown to select the target (another section, another question, "Continue to next question", or "Submit form")
  - For `text`, `textarea`, and `info_block` types, a similar dropdown defines where to go after the element
- **Required Toggle**: A switch allows marking questions as mandatory
- **ID Generation**: Unique IDs (`section_X`, `q_X`) are auto-generated to prevent collisions

### JSON Generation

When "Generate JSON" is clicked, the builder flattens the `formSections` structure into a single array of elements:

- Section headers (`type: "section_header"`) are included in this flat list, using the section's ID and name
- Questions are listed after their respective section headers, retaining their properties
- A `default_next_element_id` is calculated for each element to define the linear flow if no explicit branching occurs

### JSON Import (File/Paste)

The builder can parse a flat JSON (like the one it generates):

- It reconstructs the `formSections` data structure:
  - `section_header` types in the JSON create new sections
  - Subsequent elements are added to the most recently created section
- ID counters are updated to avoid collisions with imported IDs
- The UI is then re-rendered based on the imported data

### Test Form Feature

- Uses the generated flat JSON to simulate the end-user experience in a modal
- Renders one element at a time
- Follows `branch_to_element_id` and `default_next_element_id` for navigation
- Collects answers and displays a summary upon reaching `_SUBMIT_FORM_`
- Visually indicates required fields but does not currently enforce them during the test

## 2. Live Form Renderer - How it Works

### Loading Definition

- Attempts to fetch `form-definition.json` from its directory
- If not found, it falls back to an embedded default JSON string
- If loading fails, it displays a "no questions yet" message
- A "Dev Load" button allows manually loading a new JSON via file or paste for testing

### Core Data Structures

- **`formDefinitionData`**: Stores the flat array of form elements loaded from JSON
- **`formAnswers`**: An object to store user's answers, keyed by question ID
- **`currentElementId`**: Tracks the ID of the element currently displayed to the user

### Rendering Logic (`displayFormElement`)

Takes an `elementId` as input:

1. Finds the corresponding element object in `formDefinitionData`
2. Clears the previous element from the display area
3. Dynamically generates HTML based on the element's type:
   - **`section_header`**: Displays the label as a prominent header
   - **`info_block`**: Displays the label as informational text
   - **`text`, `textarea`, `radio`, `checkboxes`, `select`**: Renders the appropriate input field(s) along with the question label (and a required indicator `*` if `element.required === true`)

### Navigation Button

For most elements, a "Next" (or "Submit Form" if it's the last logical step) button is rendered. The target of this button is determined by:

1. The element's own `branch_to_element_id` (if defined, e.g., for text inputs)
2. The element's `default_next_element_id` (from the JSON)
3. If neither is present, the next element in the `formDefinitionData` array

### Branching and Flow

- **Radio/Select**: When an option is selected:
  - If the option has a `branch_to_element_id`, the form immediately navigates to that target after capturing and validating the current answer
  - If the option has no specific branch, the flow proceeds to the `default_next_element_id` of the current question or the next element in sequence
- **Other types** (`Text`, `Textarea`, `Info Block`, `Section Header`, `Checkboxes`): User clicks the "Next" button

### Answer Collection & Validation (`captureAndValidateCurrentAnswer`)

Called before navigating away from a question:

- Retrieves the value from the current input field(s)
- Stores the answer in the `formAnswers` object
- If the question is required and no answer is provided, it displays an error message and prevents navigation

### Form Submission (`_SUBMIT_FORM_`)

When an element or option branches to `_SUBMIT_FORM_`, or the end of the form is reached:

- The `displayAnswerSummary` function is called
- It iterates through `formAnswers` and displays a summary of all collected question labels and their corresponding answers

## II. To-Do List & Considerations

### Form Builder Enhancements

- **Robust ID Management**: While unique IDs are generated, more advanced handling for ID changes (e.g., automatically updating all references if an ID is edited, though making IDs read-only after creation simplifies this)
- **Drag-and-Drop Reordering**: Allow reordering of sections (tabs) and questions within sections
- **Conditional Logic for Showing/Hiding Questions (Builder UI)**: Add UI to define `display_condition` for questions (e.g., "Show this question if Q_X has answer Y")
- **Auto-generation of Follow-up Questions**: When an option is set to branch, offer to auto-create a new placeholder question in the target section
- **Visual Cues for Branching**: Better visual indication in the builder of where options branch to
- **Validation Rules**: Allow defining more complex validation rules beyond just "required" (e.g., email format, number range)

### Live Form Renderer Enhancements

- **Full "Required" Field Enforcement**: The test form currently only indicates required fields. The live renderer should strictly prevent proceeding if a required field is skipped (the `captureAndValidateCurrentAnswer` function does this, so it's mostly about ensuring it's called correctly before any navigation)
- **Styling and Theming**: More advanced styling options
- **Data Submission**: Implement actual data submission to a backend or other storage
- **Progress Indicator**: Display progress through the form (e.g., "Step 2 of 5")
- **Handling of Checkbox Branching**: Decide on a clear logic for how branching should work if multiple checkboxes (each potentially with its own branch target) are selected. Currently, the "Next" button handles flow for checkboxes

### Data & Content

#### List of Countries (Hague/Non-Hague)

- Integrate a comprehensive and up-to-date list of Hague Convention signatory countries
- This is crucial for the empathetic follow-up in the initial questionnaire phase
- Consider how to manage updates to this list

#### Languages Spoken by Children

- This data point is collected in the "Keepsake Box" branch
- Consider how this information will be used to customize the keepsake or time capsule clues
- Potentially offer pre-defined language lists or allow free-text input

#### Therapeutic Writing Prompts

- Develop a set of sensitive and helpful writing prompts for the time capsule, possibly categorized by the parent's situation (bereavement, separation, abduction type)

### General

- **Error Handling**: More robust error handling throughout both applications
- **Accessibility (a11y)**: Ensure both the builder and the renderer are accessible
- **Internationalization (i18n)**: If the tools need to support multiple languages for the UI itself

---

This document should serve as a good reference for the current state and future direction of the project.