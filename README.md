# begem
A bash tool for customizing queries and formatting responses automatically with gemini. Requires a gemini API key.

Detailed Documentation for begemcolor.sh (Version 13)

This document provides a comprehensive overview of the begemcolor.sh Bash script, detailing its core features, configuration options, and functions. This script is designed to facilitate interactive conversations with the Gemini API directly from your terminal, offering a rich set of features for customization, logging, and advanced querying.
1. Introduction

begemcolor.sh is a powerful and highly customizable Bash script that acts as a command-line interface for the Gemini API. It allows users to engage in conversational AI interactions, manage API keys securely, and control various aspects of the interaction, including response formatting, conversation history, and advanced multi-turn queries. Version 13 introduces significant enhancements in persistence, query stacking, and a visually engaging cyberpunk-themed interface.
2. Core Features

The script is built around several key features that enhance its utility and user experience:
2.1. Gemini API Interaction

    Conversational Responses: The primary function is to interact with the Gemini API to generate conversational AI responses based on user input.

    Model Selection: Uses gemini-1.5-flash-latest as the default model and gemini-1.0-pro as a fallback in case of quota exhaustion or model overload.

2.2. Prefix Files

    Customizable Prompts: Supports the use of prefix files (e.g., text-prefix.txt, json-prefix.txt) to prepend custom instructions or context to regular queries. This allows users to define specific personas or response formats for Gemini.

    Dynamic Selection: Users can select different prefix files from a menu, which can automatically configure autoexport settings based on the chosen prefix (e.g., selecting html-prefix.txt sets autoexport to .html).

2.3. Autoexport Responses

    Flexible Export Formats: Automatically exports Gemini responses to various file types:

        .txt (default for plain text)

        .html (for basic HTML formatted responses)

        .ps1 (for PowerShell scripts, exported as plain text)

        .json (for JSON data, with validation and pretty-printing)

    Daily Directories: Exports are organized into daily directories within begem/autoexport/YYYY-MM-DD.

    Randomized Filenames: Each exported file gets a unique filename based on timestamp and a random number to prevent overwrites.

2.4. Conversation Logging

    Daily Logs: Logs all conversation history (user queries and Gemini responses) to daily log files in begem/conversation_YYYY-MM-DD.log.

    Optional Session History: Can optionally log conversation history to history/YYYY-MM-DD/history.txt for a dedicated session transcript.

2.5. Preferences File

    Initial Prompt Customization: Supports a geminipref.txt file to define an initial prompt or persistent context that can be sent with the very first query of a session.

    Persistence Prefix: The content of geminipref.txt can also be used as the PERSISTENCE_PREFIX for the Persistence Mode.

2.6. Persistent Configuration

    Save/Load Settings: Saves and loads current script settings (selected prefix, autoexport status, history, preferences, and persistence flags) to/from lastconfig.txt. This ensures your preferred configuration persists across sessions.

2.7. Stack 5 Query (New Feature)

    Iterative Elaboration: This powerful feature allows users to iteratively query Gemini 5 times, with each subsequent query building upon and elaborating on the previous response.

    User Control: After each iteration, the user is prompted to either resend the same "keep going and elaborate" prompt or adjust the query for more specific direction.

    API Overload Handling: Includes a delay between queries and retry logic for 429 (Too Many Requests) and 503 (Service Unavailable) errors to prevent API overload.

    Model Fallback: Automatically switches to a fallback model (gemini-1.0-pro) if the primary model (gemini-1.5-flash-latest) hits a quota limit.

    Autoexport Final Result: The final, consolidated result of the 5-stack query is automatically exported based on the current autoexport settings.

2.8. Persistence Mode (New Feature)

    Contextual Conversations: When enabled, this mode prefixes all user queries with a defined persistence prefix (either from geminipref.txt or a default) and appends each user query with a timestamp to begem/persist/persisting-transcript.txt.

    Full Transcript Context: The entire persisting-transcript.txt (wrapped with special markers) is sent to Gemini with each query, providing a comprehensive historical context for the AI's responses.

    Submenu for Control: A dedicated submenu allows users to:

        Toggle Persistence on/off.

        Wipe the persistence transcript.

        Save the current PERSISTENCE_PREFIX to the preferences file (geminipref.txt).

2.9. Cyberpunk Color Theme (New Feature)

    Enhanced Terminal Interface: The script utilizes a vibrant cyberpunk aesthetic for its menus and output, improving readability and user engagement.

    Color Scheme:

        Neon Green: Configuration values, successful messages, file paths.

        Neon Pink: Option labels, specific details, version/colormode in metaheader.

        White: User prompts, general text.

        Cyan: Separators, metaheader boundaries.

        Red: Error messages, warnings.

2.10. API Key Handling

    Secure Storage: Prompts the user to paste their Gemini API key if gemini_api_key.txt is missing or invalid. The key is saved to $HOME/gemini_api_key.txt.

    Basic Validation: Performs a basic check to ensure the provided key is not empty or just whitespace.

    Exit on No Key: The script exits if no valid API key is provided.

2.11. Metaheader (New Feature)

    Branding and Versioning: Displays a custom metaheader (Elton B. - boehnenelton2024@gmail.com === version 91 colormode - begem-colors.sh ===) above the main and persistence menus.

    Visual Appeal: Uses neon green for name/email, neon pink for version/colormode, and cyan for separators, enhancing the futuristic look.

    Clean Interface: The terminal is cleared on startup for a fresh display.

3. Configuration Variables

The script uses several global variables for configuration:

    NEON_GREEN, NEON_PINK, WHITE, CYAN, RED, RESET: ANSI escape codes for terminal colors, defining the cyberpunk theme.

    METAHEADER: The formatted string displayed at the top of menus.

    SCRIPT_DIR: The absolute path to the directory where the script is located.

    GEMINI_KEY_FILE: Path to the file storing the Gemini API key (defaults to $HOME/gemini_api_key.txt).

    BEGEM_HISTORY_DIR: Base directory for all script-related files (logs, configs, stacks, persistence). Defaults to SCRIPT_DIR/begem.

    LAST_CONFIG_FILE: Path to the file where current settings are saved and loaded (BEGEM_HISTORY_DIR/lastconfig.txt).

    STACK_DIR: Directory for temporary files used by the Stack 5 Query (BEGEM_HISTORY_DIR/stack).

    PERSIST_DIR: Directory for persistence mode files (BEGEM_HISTORY_DIR/persist).

    PERSIST_TRANSCRIPT_FILE: File storing the conversation transcript for persistence mode (PERSIST_DIR/persisting-transcript.txt).

    PREFIXES_DIR: Directory where prefix .txt files are stored (BEGEM_HISTORY_DIR/prefixes).

    DEFAULT_PREFIX_FILE: Default prefix file (PREFIXES_DIR/default.txt).

    TEXT_PREFIX_FILE: Default text prefix file (PREFIXES_DIR/text-prefix.txt).

    SELECTED_PREFIX_FILENAME: Stores the filename of the currently selected prefix.

    SELECTED_PREFIX_CONTENT: Stores the actual content of the currently selected prefix.

    EXPORTS_BASE_DIR: Base directory for autoexported files (BEGEM_HISTORY_DIR/autoexport).

    AUTOEXPORT_ENABLED: Flag (true/false) to enable/disable autoexport.

    AUTOEXPORT_FILE_EXT: The file extension for autoexported files (e.g., txt, html, ps1, json).

    AUTOEXPORT_FORMAT_FUNC: The name of the function used to format the response before export (e.g., _format_txt_response).

    HISTORY_BASE_DIR: Base directory for conversation history logs (BEGEM_HISTORY_DIR/history).

    ENABLE_HISTORY: Flag (true/false) to enable/disable logging to history.txt.

    CURRENT_SESSION_HISTORY_FILE: Path to the current session's history.txt file.

    PREF_FILE: Path to the preferences file (BEGEM_HISTORY_DIR/geminipref.txt).

    ENABLE_PREFERENCES: Flag (true/false) to enable/disable the use of the preferences file.

    PREFERENCES_TEXT: Stores the content of the preferences file.

    IS_FIRST_QUERY: Flag (true/false) to track if the current query is the first in the session (used for preferences).

    ENABLE_PERSISTENCE: Flag (true/false) to enable/disable persistence mode.

    PERSISTENCE_PREFIX: The default prefix used in persistence mode if geminipref.txt is not enabled.

    GEMINI_API_BASE_URL: The base URL for the Gemini API.

    GEMINI_MODEL: The primary Gemini model to use.

    FALLBACK_MODEL: The fallback Gemini model for quota/overload issues.

    CONVERSATION_HISTORY: A JSON array string that stores the conversation history (user and model turns) for sending context to the Gemini API.

4. Functions

The script is modularized into several functions, each responsible for a specific task:
4.1. read_api_key()

    Purpose: Reads the Gemini API key from GEMINI_KEY_FILE. If the file is missing or empty, it prompts the user to paste a new key, saves it, and then loads it.

    Error Handling: Exits the script if no valid key is provided or if saving fails.

4.2. log_conversation(log_type, message)

    Purpose: Logs messages to a daily log file (begem/conversation_YYYY-MM-DD.log) and also prints them to the console.

    Arguments:

        log_type: A string indicating the type of log (e.g., "System", "You", "Gemini", "ERROR").

        message: The content of the message to log.

4.3. check_quota(model)

    Purpose: Attempts to perform a token count request to the Gemini API to check for quota issues for a given model.

    Arguments:

        model: The Gemini model to check (e.g., gemini-1.5-flash-latest).

    Return: Returns 0 on success (quota check appears okay) and 1 on failure (e.g., API error, quota exceeded).

4.4. configure_persistence()

    Purpose: Presents a submenu for managing the Persistence feature.

    Options:

        Toggle persistence on/off.

        Wipe the persisting-transcript.txt file.

        Save the current PERSISTENCE_PREFIX to geminipref.txt and enable preferences.

4.5. _stack_query()

    Purpose: Implements the "Stack 5 Query" feature, allowing for iterative, multi-turn conversations with Gemini.

    Flow:

        Prompts for an initial query.

        Sends the initial query to Gemini.

        For 5 iterations:

            Prompts the user to resend the same elaboration prompt or provide a new, adjusted query.

            Sends the updated query (including previous response) to Gemini.

            Appends the new response to a temporary file.

        Handles API errors, quota exhaustion (with fallback model), and model overload (with retries and backoff).

        Autoexports the final consolidated content from the temporary file.

4.6. query_gemini(user_prompt, model)

    Purpose: The core function for sending a prompt to the Gemini API and receiving a response.

    Arguments:

        user_prompt: The text prompt to send to Gemini.

        model: The Gemini model to use for the request.

    Functionality:

        Applies PREFERENCES_TEXT if ENABLE_PREFERENCES is true and it's the IS_FIRST_QUERY.

        Constructs the JSON payload including the CONVERSATION_HISTORY for context.

        Makes a curl POST request to the Gemini API.

        Parses the JSON response and extracts the text content.

        Updates CONVERSATION_HISTORY with both the user's prompt and Gemini's response for subsequent turns.

        Handles API errors and returns an error JSON if the response is malformed or contains an error.

4.7. _format_txt_response(raw_text)

    Purpose: Formats raw Gemini text for plain text export.

    Functionality: Strips markdown code fences (e.g., bash`, powershell`, ```) from the response.

4.8. _format_html_response(raw_text)

    Purpose: Formats raw Gemini text into a basic HTML page for export.

    Functionality: Escapes HTML characters (&, <, >, ", ') and replaces newlines with <br> tags. Includes a basic HTML structure and can link to a default.css if present.

4.9. _format_json_response(raw_text)

    Purpose: Formats raw Gemini text into pretty-printed JSON for export.

    Functionality: Strips markdown JSON code fences, validates the JSON content using jq, and pretty-prints it. If the content is not valid JSON, it wraps it as a string in a JSON object.

4.10. save_config()

    Purpose: Saves the current script configuration (prefix, autoexport, history, preferences, persistence settings) to lastconfig.txt.

4.11. load_config()

    Purpose: Loads previously saved configuration from lastconfig.txt.

    User Interaction: Prompts the user to confirm if they want to load previous settings or start with defaults.

    Validation: Checks if loaded prefix/preferences files exist; if not, it reverts to defaults or disables the feature.

4.12. show_main_config_menu()

    Purpose: Displays the main configuration menu, showing current settings and providing options to change them or start a conversation.

    Navigation: Allows users to access sub-menus for prefix selection, autoexport, history, preferences, persistence, save settings, start conversation, or exit.

4.13. select_prefix()

    Purpose: Allows the user to select, create, or delete prefix .txt files.

    Dynamic Autoexport: Automatically adjusts AUTOEXPORT_FILE_EXT and AUTOEXPORT_FORMAT_FUNC based on the selected prefix filename (e.g., html-prefix.txt sets autoexport to .html).

4.14. configure_autoexport()

    Purpose: Guides the user through enabling/disabling autoexport and selecting the desired export file type (.txt, .html, .ps1, .json).

4.15. configure_history()

    Purpose: Toggles the ENABLE_HISTORY flag, controlling whether conversation history is logged to history/YYYY-MM-DD/history.txt.

4.16. configure_preferences()

    Purpose: Toggles the ENABLE_PREFERENCES flag, controlling whether the content of geminipref.txt is used as an initial prompt. It also creates a default preferences file if one doesn't exist.

4.17. _make_prefix_file()

    Purpose: Guides the user through creating a new custom prefix .txt file, prompting for a filename and content.

4.18. _delete_prefix_file()

    Purpose: Allows the user to select and delete an existing prefix .txt file. Includes a confirmation step and prevents deletion of default.txt.

5. Usage Instructions

To use the begemcolor.sh script:

    Save the script: Copy the script content and save it as begemcolor.sh in your Termux environment.

    Make it executable: chmod +x begemcolor.sh

    Run the script: ./begemcolor.sh

    Initial Setup: On the first run, the script will prompt you for your Gemini API key. Paste it and press Enter. It will then ask if you want to load previous settings (if lastconfig.txt exists) or present the main configuration menu.

    Navigate Menus: Use the provided options (single characters like Q, P, A, H, F, G, S, X, E) to configure features or start a conversation.

    Start Conversation: Select X from the main menu to begin interacting with Gemini. Type your queries and press Enter.

    Exit: Type exit and press Enter during a conversation to quit the script. Your current settings will be saved.

6. Conclusion

begemcolor.sh version 13 is a comprehensive and user-friendly tool for interacting with the Gemini API in Termux. Its robust features for secure API key handling, flexible output formatting, detailed logging, and advanced query capabilities like "Stack 5 Query" and "Persistence Mode," combined with an engaging cyberpunk-themed interface, make it an excellent choice for developers and enthusiasts looking to leverage AI directly from their mobile terminal.
