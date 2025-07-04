#!/bin/bash

# begem.sh - Version 13
# Conversation Link: https://grok.com/chat/65ca4880-c4e5-4826-8f74-e71605f96653
# Core Features:
# - Interacts with the Gemini API to generate conversational responses.
# - Supports prefix files (e.g., text-prefix.txt, json-prefix.txt) to customize prompts for regular queries.
# - Autoexports responses to .txt (default), .html, .ps1, or .json files in begem/autoexport/YYYY-MM-DD.
# - Logs conversation history to daily files and optionally to history/YYYY-MM-DD/history.txt.
# - Supports a preferences file (geminipref.txt) for initial prompt customization and persistence prefix.
# - Saves and loads settings via lastconfig.txt for persistent configuration.
# - Stack 5 Query: Iteratively queries Gemini 5 times, elaborating on the previous response in begem/stack/temp with a delay to prevent API overload, using a fixed "keep going and elaborate" prompt (prefixes disabled) for resend, or "Expand on that further..." for adjusted queries, prompts user to resend or adjust query after each iteration, and autoexports the final result. Includes retry logic for 429/503 errors and model fallback for quota exhaustion.
# - Persistence Mode: When enabled, prefixes all queries with the preferences file content (if enabled) or "Gemini, responding in only one liners when dealing with code, or single paragraphs, give me your response to this:", appends each user query with a timestamp to begem/persist/persisting-transcript.txt, and sends the full transcript (prefixed with "[*[*[*!! historical transcription !!! *]*]*]*" and suffixed with "[*[*[*!! end of historical transcription !!! *]*]*]*") followed by the prefix and query to Gemini for context. Includes a submenu to toggle persistence, wipe the transcript, or save the persistence prefix to the preferences file.
# - Cyberpunk Color Theme: Menus use a cyberpunk aesthetic with neon green for configuration values, neon pink for option labels and specifics, white for prompts, cyan for separators, and red for errors, enhancing the terminal interface with a futuristic look.
# - API Key Handling: If the API key file ($HOME/gemini_api_key.txt) is missing or invalid, prompts the user to paste a new key, saves it to $HOME/gemini_api_key.txt, and proceeds. Exits if no key is provided.
# - Metaheader: Displays "Elton B. - boehnenelton2024@gmail.com === version 91 colormode - begem-colors.sh ===" above the main and persistence menus, with neon green for name/email, neon pink for version/colormode, and cyan for separators. Terminal clears on startup.

# --- Color Theme Configuration ---
NEON_GREEN="\033[0;92m"
NEON_PINK="\033[1;95m"
WHITE="\033[1;97m"
CYAN="\033[0;96m"
RED="\033[0;91m"
RESET="\033[0m"

# --- Metaheader Configuration ---
METAHEADER="${CYAN}=== ${NEON_GREEN}Elton B. - boehnenelton2024@gmail.com${CYAN} === ${NEON_PINK}version 91 colormode - begem-colors.sh${CYAN} ===${RESET}"

# --- Configuration ---
# Get the absolute path of the directory where the script (begem.sh) is located.
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" &>/dev/null && pwd)"

# Define the ABSOLUTE PATH to the file that contains your Gemini API key.
GEMINI_KEY_FILE="$HOME/gemini_api_key.txt" # Default API key file in user's home directory

# Directory to store conversation history, logs, stack, and persistence files
BEGEM_HISTORY_DIR="${SCRIPT_DIR}/begem"
LAST_CONFIG_FILE="${BEGEM_HISTORY_DIR}/lastconfig.txt" # File to save/load settings
STACK_DIR="${BEGEM_HISTORY_DIR}/stack" # Directory for stack query temp file
PERSIST_DIR="${BEGEM_HISTORY_DIR}/persist" # Directory for persistence transcript
PERSIST_TRANSCRIPT_FILE="${PERSIST_DIR}/persisting-transcript.txt" # Persistence transcript file

# Directory for prefix files
PREFIXES_DIR="${BEGEM_HISTORY_DIR}/prefixes"
DEFAULT_PREFIX_FILE="${PREFIXES_DIR}/default.txt"
TEXT_PREFIX_FILE="${PREFIXES_DIR}/text-prefix.txt"
SELECTED_PREFIX_FILENAME="text-prefix.txt" # Global: default to text-prefix.txt
SELECTED_PREFIX_CONTENT=""  # Global: stores the actual content of the selected prefix

# --- Autoexport Feature Configuration ---
EXPORTS_BASE_DIR="${BEGEM_HISTORY_DIR}/autoexport"
AUTOEXPORT_ENABLED="true"   # Flag: default to enabled for .txt
AUTOEXPORT_FILE_EXT="txt"   # Store the user-chosen file extension (default to txt)
AUTOEXPORT_FORMAT_FUNC="_format_txt_response"  # Default to plain text formatting
# --- End Autoexport Configuration ---

# --- History Feature Configuration ---
HISTORY_BASE_DIR="${BEGEM_HISTORY_DIR}/history"
ENABLE_HISTORY="false"             # Flag: "true" or "false"
CURRENT_SESSION_HISTORY_FILE=""    # Stores the full path to the daily history.txt file
# --- END History Configuration ---

# --- Preferences File Configuration ---
PREF_FILE="${BEGEM_HISTORY_DIR}/geminipref.txt"
ENABLE_PREFERENCES="false"         # Flag: "true" or "false"
PREFERENCES_TEXT=""                # Global: stores the content of the preferences file
IS_FIRST_QUERY="true"              # Flag: used to submit preferences only with the very first query
# --- END Preferences Configuration ---

# --- Persistence Configuration ---
ENABLE_PERSISTENCE="false"         # Flag: "true" or "false"
PERSISTENCE_PREFIX="Gemini, responding in only one liners when dealing with code, or single paragraphs, give me your response to this:"
# --- END Persistence Configuration ---

# Gemini API Endpoint and Model Fallback
GEMINI_API_BASE_URL="https://generativelanguage.googleapis.com/v1beta/models"
GEMINI_MODEL="gemini-1.5-flash-latest" # Default model
FALLBACK_MODEL="gemini-1.0-pro"       # Fallback for quota exhaustion

# Global variable to store conversation history for context in API calls
CONVERSATION_HISTORY="[]"

# --- Functions ---

# Function to read Gemini API Key from file or prompt user
read_api_key() {
    if [ ! -f "${GEMINI_KEY_FILE}" ] || [ -z "$(cat "${GEMINI_KEY_FILE}" | tr -d '[:space:]')" ]; then
        echo -e "${RED}Gemini API key file not found or invalid at '${NEON_GREEN}${GEMINI_KEY_FILE}${RED}'.${RESET}"
        echo -e "${WHITE}Please paste your Gemini API key (or press Enter to exit): ${RESET}\c"
        read new_api_key
        if [ -z "${new_api_key}" ]; then
            log_conversation "ERROR" "No API key provided. Exiting script."
            echo -e "${RED}No API key provided. Exiting.${RESET}" | tee /dev/stderr
            exit 1
        fi
        # Basic validation: ensure key is non-empty and not just whitespace
        if [[ "${new_api_key}" =~ ^[[:space:]]*$ ]]; then
            log_conversation "ERROR" "Invalid API key provided (empty or whitespace). Exiting script."
            echo -e "${RED}Invalid API key (empty or whitespace). Exiting.${RESET}" | tee /dev/stderr
            exit 1
        fi
        # Save the new key to GEMINI_KEY_FILE
        echo "${new_api_key}" > "${GEMINI_KEY_FILE}"
        if [ $? -eq 0 ]; then
            log_conversation "System" "New API key saved to '${GEMINI_KEY_FILE}'."
            echo -e "${WHITE}API key saved to '${NEON_GREEN}${GEMINI_KEY_FILE}${RESET}'."
        else
            log_conversation "ERROR" "Failed to save API key to '${GEMINI_KEY_FILE}'. Check permissions."
            echo -e "${RED}Failed to save API key to '${GEMINI_KEY_FILE}'. Check permissions.${RESET}" | tee /dev/stderr
            exit 1
        fi
    fi
    GEMINI_API_KEY=$(head -n 1 "${GEMINI_KEY_FILE}" | tr -d '[:space:]')
    if [ -z "${GEMINI_API_KEY}" ]; then
        log_conversation "ERROR" "Gemini API key is empty in '${GEMINI_KEY_FILE}'. Exiting script."
        echo -e "${RED}ERROR: Gemini API key is empty in '${NEON_GREEN}${GEMINI_KEY_FILE}${RED}'. Please provide a valid key.${RESET}" | tee /dev/stderr
        exit 1
    fi
    export GEMINI_API_KEY
}

# Function to log conversation history to daily file and console
log_conversation() {
    local log_type="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local log_file="${BEGEM_HISTORY_DIR}/conversation_$(date +%Y-%m-%d).log"

    mkdir -p "${BEGEM_HISTORY_DIR}"
    echo "[$timestamp] $log_type: $message" | tee -a "${log_file}"
}

# Function to check quota usage
check_quota() {
    local model="$1"
    local api_url="${GEMINI_API_BASE_URL}/${model}:countTokens?key=${GEMINI_API_KEY}"
    local sample_prompt="Sample quota check prompt"
    local json_payload=$(jq -n --arg prompt "$sample_prompt" '{contents: [{"role": "user", "parts": [{"text": $prompt}]}]}')

    local response=$(curl -s -X POST -H "Content-Type: application/json" "${api_url}" -d "${json_payload}")
    if echo "${response}" | jq -e '.error' >/dev/null; then
        local error_message=$(echo "${response}" | jq -r '.error.message // "Unknown quota check error"')
        log_conversation "WARNING" "Quota check failed: ${error_message}"
        echo -e "${RED}Warning: Unable to verify quota. Proceeding, but you may hit limits.${RESET}"
        return 1
    fi
    log_conversation "System" "Quota check performed for model '${model}'."
    return 0
}

# --- Persistence Configuration Menu ---
configure_persistence() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Configuring Persistence feature..."

    echo -e "${METAHEADER}\n"
    echo -e "${CYAN}=== Persistence Configuration ===${RESET}"
    echo -e "${NEON_PINK}1) Toggle Persistence (Currently: ${NEON_GREEN}${ENABLE_PERSISTENCE}${RESET}${NEON_PINK})${RESET}"
    echo -e "${NEON_PINK}2) Wipe Persistence Transcript${RESET}"
    echo -e "${NEON_PINK}3) Save Persistence Prefix to Preferences File${RESET}"
    echo -e "${NEON_PINK}4) Back to main menu${RESET}"
    echo -e "${CYAN}===============================${RESET}"
    echo -e "${WHITE}Choose an option (1-4): ${RESET}\c"
    local choice
    read choice

    case "${choice}" in
        1)
            if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
                ENABLE_PERSISTENCE="false"
                echo -e "${WHITE}Persistence is now: ${NEON_GREEN}DISABLED${RESET}."
                log_conversation "System" "Persistence disabled."
            else
                ENABLE_PERSISTENCE="true"
                echo -e "${WHITE}Persistence is now: ${NEON_GREEN}ENABLED${RESET}."
                log_conversation "System" "Persistence enabled."
            fi
            ;;
        2)
            if [ -f "${PERSIST_TRANSCRIPT_FILE}" ]; then
                local confirm
                echo -e "${WHITE}Are you sure you want to wipe '${NEON_GREEN}${PERSIST_TRANSCRIPT_FILE}${RESET}'? (y/N): ${RESET}\c"
                read confirm
                if [[ "${confirm,,}" == "y" ]]; then
                    : > "${PERSIST_TRANSCRIPT_FILE}"
                    if [ $? -eq 0 ]; then
                        log_conversation "System" "Persistence transcript '${PERSIST_TRANSCRIPT_FILE}' wiped."
                        echo -e "${WHITE}Transcript wiped.${RESET}"
                    else
                        log_conversation "ERROR" "Failed to wipe persistence transcript '${PERSIST_TRANSCRIPT_FILE}'."
                        echo -e "${RED}Failed to wipe transcript.${RESET}"
                    fi
                else
                    echo -e "${WHITE}Wipe cancelled.${RESET}"
                fi
            else
                echo -e "${WHITE}No transcript file exists to wipe.${RESET}"
                log_conversation "System" "No persistence transcript found to wipe."
            fi
            ;;
        3)
            mkdir -p "${BEGEM_HISTORY_DIR}"
            echo "${PERSISTENCE_PREFIX}" > "${PREF_FILE}"
            if [ $? -eq 0 ]; then
                ENABLE_PREFERENCES="true"
                PREFERENCES_TEXT="${PERSISTENCE_PREFIX}"
                log_conversation "System" "Persistence prefix saved to '${PREF_FILE}' and preferences enabled."
                echo -e "${WHITE}Persistence prefix saved to preferences file and enabled.${RESET}"
            else
                log_conversation "ERROR" "Failed to save persistence prefix to '${PREF_FILE}'."
                echo -e "${RED}Failed to save to preferences file.${RESET}"
            fi
            ;;
        4)
            return
            ;;
        *)
            echo -e "${RED}Invalid choice. Please enter 1, 2, 3, or 4.${RESET}"
            ;;
    esac
    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

# --- Stack 5 Query ---
_stack_query() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Starting Stack 5 Query..."

    mkdir -p "${STACK_DIR}" || { log_conversation "ERROR" "Failed to create stack directory: '${STACK_DIR}'. Check permissions."; echo -e "${RED}Failed to create stack directory.${RESET}"; read -p "Press Enter to continue..."; return; }
    mkdir -p "${PERSIST_DIR}" || { log_conversation "ERROR" "Failed to create persistence directory: '${PERSIST_DIR}'. Check permissions."; echo -e "${RED}Failed to create persistence directory.${RESET}"; read -p "Press Enter to continue..."; return; }

    local temp_file="${STACK_DIR}/temp"
    local user_query
    echo -e "${WHITE}Enter your initial query for Stack 5: ${RESET}\c"
    read user_query
    if [ -z "$user_query" ]; then
        log_conversation "System" "Stack 5 Query cancelled: Empty query."
        echo -e "${RED}Query cannot be empty.${RESET}"
        read -p "Press Enter to continue..."
        return
    fi

    log_conversation "You" "Stack 5 Initial Query: ${user_query}"

    : > "${temp_file}" || { log_conversation "ERROR" "Failed to clear temp file: '${temp_file}'. Check permissions."; echo -e "${RED}Failed to clear temp file.${RESET}"; read -p "Press Enter to continue..."; return; }

    check_quota "${GEMINI_MODEL}"
    if [ $? -ne 0 ]; then
        echo -e "${RED}Warning: Unable to verify quota. Proceeding, but you may hit limits.${RESET}"
    fi

    local current_model="${GEMINI_MODEL}"
    local retry_count=0
    local max_retries=3
    local initial_response=""
    local query_to_send="${user_query}"
    local persistence_prefix="${PERSISTENCE_PREFIX}"
    if [ "${ENABLE_PREFERENCES}" = "true" ] && [ -n "${PREFERENCES_TEXT}" ]; then
        persistence_prefix="${PREFERENCES_TEXT}"
    fi
    if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        echo "[$timestamp] You: ${user_query}" >> "${PERSIST_TRANSCRIPT_FILE}"
        local transcript_content=""
        [ -f "${PERSIST_TRANSCRIPT_FILE}" ] && transcript_content=$(cat "${PERSIST_TRANSCRIPT_FILE}")
        query_to_send="[*[*[*!! historical transcription !!! *]*]*]*\n${transcript_content}\n[*[*[*!! end of historical transcription !!! *]*]*]*\n\n${persistence_prefix}\n${user_query}"
    fi

    while [ $retry_count -lt $max_retries ]; do
        initial_response=$(query_gemini "${query_to_send}" "${current_model}")
        if [ $? -eq 0 ]; then
            break
        fi
        local error_message=$(echo "${initial_response}" | jq -r '.error.message // "Unknown error"')
        if [[ "${error_message}" =~ "You exceeded your current quota" ]]; then
            if [ "${current_model}" != "${FALLBACK_MODEL}" ]; then
                log_conversation "System" "Quota exceeded for '${current_model}'. Switching to fallback model '${FALLBACK_MODEL}'."
                echo -e "${RED}Quota exceeded. Switching to fallback model '${FALLBACK_MODEL}'.${RESET}"
                current_model="${FALLBACK_MODEL}"
                retry_count=0
                sleep 2
                continue
            else
                log_conversation "ERROR" "Quota exceeded for fallback model '${FALLBACK_MODEL}'. Aborting Stack 5."
                echo -e "${RED}Quota exceeded for all models. Check https://ai.google.dev/gemini-api/docs/rate-limits or upgrade your plan.${RESET}"
                read -p "Press Enter to continue..."
                return
            fi
        elif [[ "${error_message}" =~ "The model is overloaded" ]]; then
            retry_count=$((retry_count + 1))
            local backoff=$((2 ** retry_count))
            log_conversation "System" "Model '${current_model}' overloaded. Retrying ($retry_count/$max_retries) after ${backoff}s..."
            echo -e "${RED}Model overloaded. Retrying ($retry_count/$max_retries) after ${backoff}s...${RESET}"
            sleep "${backoff}"
        else
            log_conversation "ERROR" "Failed to get initial Stack 5 response: ${error_message}"
            echo -e "${RED}Failed to get initial response: ${error_message}${RESET}"
            read -p "Press Enter to continue..."
            return
        fi
    done
    if [ $retry_count -ge $max_retries ]; then
        log_conversation "ERROR" "Max retries reached for initial query. Aborting Stack 5."
        echo -e "${RED}Max retries reached. Model is overloaded. Try again later.${RESET}"
        read -p "Press Enter to continue..."
        return
    fi

    echo "${initial_response}" > "${temp_file}"
    log_conversation "System" "Initial Stack 5 response stored in '${temp_file}'."
    log_conversation "Gemini" "Stack 5 (0/5): ${initial_response}"

    local last_query="Keep going and elaborate in detail, creating more content based on the following:\n${initial_response}"
    for i in {1..5}; do
        clear
        echo -e "\n\n${WHITE}Creating stack ${i} out of 5 complete...${RESET}"

        if [ ! -f "${temp_file}" ] || [ ! -r "${temp_file}" ]; then
            log_conversation "ERROR" "Temp file '${temp_file}' missing or unreadable at iteration ${i}. Aborting Stack 5."
            echo -e "${RED}Error: Temp file missing or unreadable at iteration ${i}. Aborting.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        local temp_content=$(cat "${temp_file}")
        if [ -z "${temp_content}" ]; then
            log_conversation "ERROR" "Temp file '${temp_file}' is empty at iteration ${i}. Aborting Stack 5."
            echo -e "${RED}Error: Temp file is empty at iteration ${i}. Aborting.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        echo -e "${WHITE}Resend same query or adjust? (r/a, default: r): ${RESET}\c"
        read choice
        if [[ "${choice,,}" == "a" ]]; then
            clear
            echo -e "\n\n${CYAN}--- Last Response (Iteration $((i-1))) ---${RESET}"
            cat "${temp_file}"
            echo -e "${CYAN}---------------------------------------${RESET}"
            echo -e "${WHITE}Enter new query for iteration ${i}: ${RESET}\c"
            read user_query
            if [ -z "${user_query}" ]; then
                log_conversation "System" "Empty query entered for iteration ${i}. Using default elaboration prompt."
                last_query="Keep going and elaborate in detail, creating more content based on the following:\n${temp_content}"
            else
                log_conversation "You" "Stack 5 Adjusted Query (iteration ${i}/5): ${user_query}"
                last_query="Expand on that further and submit the last result to Gemini:\n${temp_content}\n\nNew query: ${user_query}"
            fi
        else
            log_conversation "System" "Resending same query for iteration ${i}/5."
            last_query="Keep going and elaborate in detail, creating more content based on the following:\n${temp_content}"
        fi

        retry_count=0
        local next_response=""
        local query_to_send="${last_query}"
        if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
            local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
            local clean_query=""
            if [[ "${last_query}" =~ ^Expand\ on\ that\ further ]]; then
                clean_query=$(echo "${last_query}" | sed -n 's/.*New query: \(.*\)/\1/p')
                [ -z "${clean_query}" ] && clean_query="(Resend) Keep going and elaborate"
                echo "[$timestamp] You: ${clean_query}" >> "${PERSIST_TRANSCRIPT_FILE}"
            else
                echo "[$timestamp] You: (Resend) Keep going and elaborate" >> "${PERSIST_TRANSCRIPT_FILE}"
            fi
            local transcript_content=""
            [ -f "${PERSIST_TRANSCRIPT_FILE}" ] && transcript_content=$(cat "${PERSIST_TRANSCRIPT_FILE}")
            query_to_send="[*[*[*!! historical transcription !!! *]*]*]*\n${transcript_content}\n[*[*[*!! end of historical transcription !!! *]*]*]*\n\n${persistence_prefix}\n${last_query}"
        fi

        while [ $retry_count -lt $max_retries ]; do
            log_conversation "System" "Stack 5 iteration ${i}/5: Sending query with temp file content using model '${current_model}'."
            sleep 1
            next_response=$(query_gemini "${query_to_send}" "${current_model}")
            if [ $? -eq 0 ]; then
                break
            fi
            local error_message=$(echo "${next_response}" | jq -r '.error.message // "Unknown error"')
            if [[ "${error_message}" =~ "You exceeded your current quota" ]]; then
                if [ "${current_model}" != "${FALLBACK_MODEL}" ]; then
                    log_conversation "System" "Quota exceeded for '${current_model}'. Switching to fallback model '${FALLBACK_MODEL}'."
                    echo -e "${RED}Quota exceeded. Switching to fallback model '${FALLBACK_MODEL}'.${RESET}"
                    current_model="${FALLBACK_MODEL}"
                    retry_count=0
                    sleep 2
                    continue
                else
                    log_conversation "ERROR" "Quota exceeded for fallback model '${FALLBACK_MODEL}' at iteration ${i}. Aborting Stack 5."
                    echo -e "${RED}Quota exceeded for all models. Check https://ai.google.dev/gemini-api/docs/rate-limits or upgrade your plan.${RESET}"
                    read -p "Press Enter to continue..."
                    return
                fi
            elif [[ "${error_message}" =~ "The model is overloaded" ]]; then
                retry_count=$((retry_count + 1))
                local backoff=$((2 ** retry_count))
                log_conversation "System" "Model '${current_model}' overloaded. Retrying ($retry_count/$max_retries) after ${backoff}s..."
                echo -e "${RED}Model overloaded. Retrying ($retry_count/$max_retries) after ${backoff}s...${RESET}"
                sleep "${backoff}"
            else
                log_conversation "ERROR" "Failed to get Stack 5 response at iteration ${i}: ${error_message}"
                echo -e "${RED}Failed to get response at iteration ${i}: ${error_message}${RESET}"
                read -p "Press Enter to continue..."
                return
            fi
        done
        if [ $retry_count -ge $max_retries ]; then
            log_conversation "ERROR" "Max retries reached for iteration ${i}. Aborting Stack 5."
            echo -e "${RED}Max retries reached. Model is overloaded. Try again later.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        echo -e "\n${CYAN}--- Iteration ${i} ---${RESET}\n${next_response}" >> "${temp_file}"
        log_conversation "System" "Appended Stack 5 response (iteration ${i}/5) to '${temp_file}'."
        log_conversation "Gemini" "Stack 5 (${i}/5): ${next_response}"
    done

    clear
    echo -e "\n\n${WHITE}Creating stack 5 out of 5 complete...${RESET}"

    if [ "${AUTOEXPORT_ENABLED}" = "true" ]; then
        local final_content=$(cat "${temp_file}")
        if [ -z "${final_content}" ]; then
            log_conversation "ERROR" "Final Stack 5 content is empty. Skipping export."
            echo -e "${RED}Error: Final Stack 5 content is empty.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        local current_date=$(date +%Y-%m-%d)
        local daily_export_dir="${EXPORTS_BASE_DIR}/${current_date}"
        mkdir -p "${daily_export_dir}" || { log_conversation "ERROR" "Failed to create daily export directory: '${daily_export_dir}'. Check permissions."; echo -e "${RED}Failed to create export directory.${RESET}"; read -p "Press Enter to continue..."; return; }

        local export_filename="stack_$(date +%H%M%S)_$(( RANDOM % 1000 )).${AUTOEXPORT_FILE_EXT}"
        local export_filepath="${daily_export_dir}/${export_filename}"
        log_conversation "System" "Attempting to export Stack 5 result to '${export_filepath}'"

        if [ -z "${AUTOEXPORT_FILE_EXT}" ]; then
            log_conversation "ERROR" "AUTOEXPORT_FILE_EXT is empty. Skipping Stack 5 export."
            echo -e "${RED}Error: Autoexport file extension not set.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        if type -t "${AUTOEXPORT_FORMAT_FUNC}" &>/dev/null && [ "$(type -t "${AUTOEXPORT_FORMAT_FUNC}")" = "function" ]; then
            formatted_response=$("${AUTOEXPORT_FORMAT_FUNC}" "${final_content}")
        else
            log_conversation "ERROR" "Formatter function '${AUTOEXPORT_FORMAT_FUNC}' not found. Exporting raw Stack 5 content."
            formatted_response="${final_content}"
        fi

        if [ -z "${formatted_response}" ]; then
            log_conversation "ERROR" "Formatted Stack 5 response is empty. Skipping export to '${export_filepath}'."
            echo -e "${RED}Error: Formatted Stack 5 response is empty.${RESET}"
            read -p "Press Enter to continue..."
            return
        fi

        echo "${formatted_response}" > "${export_filepath}"
        if [ $? -eq 0 ]; then
            log_conversation "System" "Stack 5 result exported to '${export_filepath}'."
            echo -e "${WHITE}Stack 5 result exported to '${NEON_GREEN}${export_filepath}${RESET}'."
        else
            log_conversation "ERROR" "Failed to export Stack 5 result to '${export_filepath}'. Check permissions."
            echo -e "${RED}Failed to export Stack 5 result to '${export_filepath}'.${RESET}"
        fi
    else
        log_conversation "System" "Autoexport is disabled. Stack 5 result stored in '${temp_file}' but not exported."
        echo -e "${WHITE}Autoexport is disabled. Stack 5 result stored in '${NEON_GREEN}${temp_file}${RESET}'."
    fi

    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

# Function to send query to Gemini API with specified model
query_gemini() {
    local user_prompt="$1"
    local model="$2"
    local final_prompt_text="${user_prompt}"
    local persistence_prefix="${PERSISTENCE_PREFIX}"
    if [ "${ENABLE_PREFERENCES}" = "true" ] && [ -n "${PREFERENCES_TEXT}" ]; then
        persistence_prefix="${PREFERENCES_TEXT}"
    fi

    if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local clean_query="${user_prompt}"
        if [[ "${user_prompt}" =~ ^Expand\ on\ that\ further ]]; then
            clean_query=$(echo "${user_prompt}" | sed -n 's/.*New query: \(.*\)/\1/p')
            [ -z "${clean_query}" ] && clean_query="(Resend) Keep going and elaborate"
        fi
        mkdir -p "${PERSIST_DIR}"
        echo "[$timestamp] You: ${clean_query}" >> "${PERSIST_TRANSCRIPT_FILE}"
        local transcript_content=""
        [ -f "${PERSIST_TRANSCRIPT_FILE}" ] && transcript_content=$(cat "${PERSIST_TRANSCRIPT_FILE}")
        final_prompt_text="[*[*[*!! historical transcription !!! *]*]*]*\n${transcript_content}\n[*[*[*!! end of historical transcription !!! *]*]*]*\n\n${persistence_prefix}\n${user_prompt}"
    fi

    if [ "${IS_FIRST_QUERY}" = "true" ]; then
        IS_FIRST_QUERY="false"
        if [ "${ENABLE_PREFERENCES}" = "true" ] && [ "${ENABLE_PERSISTENCE}" != "true" ]; then
            final_prompt_text="${PREFERENCES_TEXT}\n${final_prompt_text}"
            log_conversation "System" "Preferences text submitted with first query."
        fi
    fi

    local api_url="${GEMINI_API_BASE_URL}/${model}:generateContent?key=${GEMINI_API_KEY}"

    CONVERSATION_HISTORY=$(echo "${CONVERSATION_HISTORY}" | jq --arg prompt "$final_prompt_text" '. + [{"role": "user", "parts": [{"text": $prompt}]}]')

    local json_payload=$(jq -n --argjson history "${CONVERSATION_HISTORY}" '{contents: $history}')

    local response=$(curl -s -X POST -H "Content-Type: application/json" \
                           "${api_url}" \
                           -d "${json_payload}")

    if echo "${response}" | jq -e '.error' >/dev/null; then
        local error_message=$(echo "${response}" | jq -r '.error.message // "Unknown API Error"')
        log_conversation "ERROR" "API Error: ${error_message}"
        echo "{\"error\": {\"message\": \"${error_message}\"}}"
        return 1
    fi

    local text_content=$(echo "${response}" | jq -r '.candidates[0].content.parts[0].text // empty')

    if [ -z "${text_content}" ]; then
        log_conversation "ERROR" "Could not extract text content from Gemini response."
        echo "{\"error\": {\"message\": \"Could not extract text content from Gemini response.\"}}"
        return 1
    fi

    CONVERSATION_HISTORY=$(echo "${CONVERSATION_HISTORY}" | jq --arg response "$text_content" '. + [{"role": "model", "parts": [{"text": $response}]}]')

    echo "${text_content}"
    return 0
}

# --- Response Formatters ---
_format_txt_response() {
    local raw_text="$1"
    local clean_text=$(echo "$raw_text" | sed '/^```bash/d' | sed '/^```powershell/d' | sed '/^```/d')
    printf "%s" "$clean_text"
}

_format_html_response() {
    local raw_text="$1"
    local escaped_html_text
    escaped_html_text=$(printf "%s" "$raw_text" | \
                        sed -e 's/&/\&/g' \
                            -e 's/</\</g' \
                            -e 's/>/\>/g' \
                            -e 's/"/\"/g' \
                            -e "s/'/\'/g")
    local br_formatted_text=$(echo "$escaped_html_text" | sed 's/\n/<br>/g')

    local css_link=""
    if [ -f "${BEGEM_HISTORY_DIR}/default.css" ]; then
        css_link="    <link rel=\"stylesheet\" type=\"text/css\" href=\"../../default.css\">"
    fi

    cat <<-EOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gemini Export</title>
${css_link}
</head>
<body>
    <b>
$(printf "%s" "$br_formatted_text")
    </b>
</body>
</html>
EOF
}

_format_json_response() {
    local raw_text="$1"
    log_conversation "System" "Processing JSON response..."
    local clean_text=$(echo "$raw_text" | sed '/^```json/d' | sed '/^```/d')
    if [ -z "$clean_text" ]; then
        log_conversation "ERROR" "JSON response is empty after stripping code fences."
        jq -n '{"error": "Empty response"}'
        return
    fi
    if echo "$clean_text" | jq -e . >/dev/null 2>&1; then
        log_conversation "System" "JSON response is valid. Formatting with jq."
        echo "$clean_text" | jq .
    else
        log_conversation "System" "JSON response is invalid. Wrapping as string."
        jq -n --arg text "$clean_text" '{ "response": $text }'
    fi
}

# --- Save/Load Configuration Functions ---
save_config() {
    log_conversation "System" "Saving current settings to '${LAST_CONFIG_FILE}'..."
    mkdir -p "${BEGEM_HISTORY_DIR}" || { log_conversation "ERROR" "Failed to create history directory for config file. Check permissions."; echo -e "${RED}Failed to create history directory for config file.${RESET}"; return 1; }

    cat <<-EOF > "${LAST_CONFIG_FILE}"
PREFIX_FILENAME=${SELECTED_PREFIX_FILENAME}
AUTOEXPORT_ENABLED=${AUTOEXPORT_ENABLED}
AUTOEXPORT_FILE_EXT=${AUTOEXPORT_FILE_EXT}
AUTOEXPORT_FORMAT_FUNC=${AUTOEXPORT_FORMAT_FUNC}
ENABLE_HISTORY=${ENABLE_HISTORY}
ENABLE_PREFERENCES=${ENABLE_PREFERENCES}
ENABLE_PERSISTENCE=${ENABLE_PERSISTENCE}
EOF
    if [ $? -eq 0 ]; then
        log_conversation "System" "Settings saved successfully to '${LAST_CONFIG_FILE}'."
        echo -e "${WHITE}Settings saved successfully to '${NEON_GREEN}${LAST_CONFIG_FILE}${RESET}'."
    else
        log_conversation "ERROR" "Failed to save settings to '${LAST_CONFIG_FILE}'."
        echo -e "${RED}Failed to save settings to '${LAST_CONFIG_FILE}'.${RESET}"
    fi
}

load_config() {
    log_conversation "System" "Checking for previous settings..."
    if [ -f "${LAST_CONFIG_FILE}" ]; then
        local temp_prefix_filename=$(grep '^PREFIX_FILENAME=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_autoexport_enabled=$(grep '^AUTOEXPORT_ENABLED=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_autoexport_file_ext=$(grep '^AUTOEXPORT_FILE_EXT=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_autoexport_format_func=$(grep '^AUTOEXPORT_FORMAT_FUNC=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_enable_history=$(grep '^ENABLE_HISTORY=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_enable_preferences=$(grep '^ENABLE_PREFERENCES=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)
        local temp_enable_persistence=$(grep '^ENABLE_PERSISTENCE=' "${LAST_CONFIG_FILE}" | cut -d'=' -f2)

        echo -e "\n${CYAN}=== Previous Settings Found ===${RESET}"
        echo -e "${NEON_PINK}Prefix:${RESET} ${NEON_GREEN}$([ -z "$temp_prefix_filename" ] && echo "None" || echo "$temp_prefix_filename")${RESET}"
        echo -e "${NEON_PINK}Autoexport:${RESET} ${NEON_GREEN}${temp_autoexport_enabled}${RESET} ${NEON_PINK}(File type: ${NEON_GREEN}${temp_autoexport_file_ext:-"N/A"}${RESET}${NEON_PINK})${RESET}"
        echo -e "${NEON_PINK}Conversation History:${RESET} ${NEON_GREEN}${temp_enable_history}${RESET}"
        echo -e "${NEON_PINK}Preferences File:${RESET} ${NEON_GREEN}${temp_enable_preferences}${RESET}"
        echo -e "${NEON_PINK}Persistence:${RESET} ${NEON_GREEN}${temp_enable_persistence}${RESET}"
        echo -e "${CYAN}===============================${RESET}"

        local load_choice
        while true; do
            echo -e "${WHITE}Load previous settings? (yes/no): ${RESET}\c"
            read load_choice
            case "${load_choice,,}" in
                yes|y)
                    SELECTED_PREFIX_FILENAME="${temp_prefix_filename}"
                    AUTOEXPORT_ENABLED="${temp_autoexport_enabled}"
                    AUTOEXPORT_FILE_EXT="${temp_autoexport_file_ext}"
                    AUTOEXPORT_FORMAT_FUNC="${temp_autoexport_format_func}"
                    ENABLE_HISTORY="${temp_enable_history}"
                    ENABLE_PREFERENCES="${temp_enable_preferences}"
                    ENABLE_PERSISTENCE="${temp_enable_persistence}"

                    if [ -n "$SELECTED_PREFIX_FILENAME" ]; then
                        if [ -f "${PREFIXES_DIR}/${SELECTED_PREFIX_FILENAME}" ]; then
                            SELECTED_PREFIX_CONTENT=$(cat "${PREFIXES_DIR}/${SELECTED_PREFIX_FILENAME}")
                            log_conversation "System" "Loaded prefix from '${SELECTED_PREFIX_FILENAME}'."
                        else
                            log_conversation "WARNING" "Loaded prefix file '${SELECTED_PREFIX_FILENAME}' not found. Reverting to 'text-prefix.txt'."
                            SELECTED_PREFIX_FILENAME="text-prefix.txt"
                            if [ -f "${TEXT_PREFIX_FILE}" ]; then
                                SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
                            else
                                echo "Provide a concise response in plain text that" > "${TEXT_PREFIX_FILE}"
                                SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
                                log_conversation "System" "Created and loaded text prefix file: '${TEXT_PREFIX_FILE}'."
                            fi
                        fi
                    fi
                    if [ "${ENABLE_PREFERENCES}" = "true" ]; then
                        if [ -f "${PREF_FILE}" ]; then
                            PREFERENCES_TEXT=$(cat "${PREF_FILE}")
                            log_conversation "System" "Loaded preferences text from '${PREF_FILE}'."
                        else
                            log_conversation "WARNING" "Preferences file '${PREF_FILE}' not found, but was enabled in config. Disabling feature."
                            ENABLE_PREFERENCES="false"
                        fi
                    fi
                    if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
                        mkdir -p "${PERSIST_DIR}" || { log_conversation "ERROR" "Failed to create persistence directory: '${PERSIST_DIR}'. Disabling persistence."; echo -e "${RED}Failed to create persistence directory.${RESET}"; ENABLE_PERSISTENCE="false"; }
                    fi

                    log_conversation "System" "Previous settings loaded."
                    return 0
                    ;;
                no|n)
                    log_conversation "System" "Previous settings ignored. Using default settings (text-prefix.txt, .txt autoexport, persistence disabled)."
                    SELECTED_PREFIX_FILENAME="text-prefix.txt"
                    if [ -f "${TEXT_PREFIX_FILE}" ]; then
                        SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
                    else
                        echo "Provide a concise response in plain text that" > "${TEXT_PREFIX_FILE}"
                        SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
                        log_conversation "System" "Created and loaded text prefix file: '${TEXT_PREFIX_FILE}'."
                    fi
                    AUTOEXPORT_ENABLED="true"
                    AUTOEXPORT_FILE_EXT="txt"
                    AUTOEXPORT_FORMAT_FUNC="_format_txt_response"
                    ENABLE_PERSISTENCE="false"
                    return 1
                    ;;
                *)
                    echo -e "${RED}Invalid choice. Please enter 'yes' or 'no'.${RESET}"
                    ;;
            esac
        done
    else
        log_conversation "System" "No previous settings found. Using default settings (text-prefix.txt, .txt autoexport, persistence disabled)."
        SELECTED_PREFIX_FILENAME="text-prefix.txt"
        if [ -f "${TEXT_PREFIX_FILE}" ]; then
            SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
        else
            echo "Provide a concise response in plain text that" > "${TEXT_PREFIX_FILE}"
            SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
            log_conversation "System" "Created and loaded text prefix file: '${TEXT_PREFIX_FILE}'."
        fi
        AUTOEXPORT_ENABLED="true"
        AUTOEXPORT_FILE_EXT="txt"
        AUTOEXPORT_FORMAT_FUNC="_format_txt_response"
        ENABLE_PERSISTENCE="false"
        return 1
    fi
}

# --- Consolidated Main Configuration Menu ---
show_main_config_menu() {
    local choice
    local done_configuring=false

    while ! "$done_configuring"; do
        clear
        echo -e "\n\n${METAHEADER}\n"
        echo -e "${CYAN}=== Current Configuration ===${RESET}"
        local display_prefix_name=$( [ -z "$SELECTED_PREFIX_FILENAME" ] && echo "None" || echo "$SELECTED_PREFIX_FILENAME" )
        local display_prefix_preview=$( [ -z "$SELECTED_PREFIX_CONTENT" ] && echo "" || head -n 1 <<< "$SELECTED_PREFIX_CONTENT" | cut -c 1-50 )
        [ -n "$display_prefix_preview" ] && display_prefix_preview=" ($display_prefix_preview...)"
        echo -e "${NEON_PINK}1) Prefix:${RESET} ${NEON_GREEN}${display_prefix_name}${display_prefix_preview}${RESET}"
        echo -e "${NEON_PINK}2) Autoexport:${RESET} ${NEON_GREEN}${AUTOEXPORT_ENABLED}${RESET} ${NEON_PINK}(File type: ${NEON_GREEN}${AUTOEXPORT_FILE_EXT:-"N/A"}${RESET}${NEON_PINK})${RESET}"
        echo -e "${NEON_PINK}3) Conversation History:${RESET} ${NEON_GREEN}${ENABLE_HISTORY}${RESET}"
        echo -e "${NEON_PINK}4) Preferences File:${RESET} ${NEON_GREEN}${ENABLE_PREFERENCES}${RESET}"
        echo -e "${NEON_PINK}5) Persistence:${RESET} ${NEON_GREEN}${ENABLE_PERSISTENCE}${RESET}"
        echo -e "${CYAN}=============================${RESET}"
        echo -e "${CYAN}=== Options ===${RESET}"
        echo -e "${NEON_PINK}Q) Stack 5 Query${RESET}"
        echo -e "${NEON_PINK}P) Select/Change Prefix${RESET}"
        echo -e "${NEON_PINK}A) Configure Autoexport${RESET}"
        echo -e "${NEON_PINK}H) Enable/Disable Conversation History${RESET}"
        echo -e "${NEON_PINK}F) Enable/Disable Preferences File${RESET}"
        echo -e "${NEON_PINK}G) Configure Persistence${RESET}"
        echo -e "${NEON_PINK}S) Save current settings for next time${RESET}"
        echo -e "${NEON_PINK}X) Start conversation with current settings${RESET}"
        echo -e "${NEON_PINK}E) Exit${RESET}"
        echo -e "${CYAN}===============${RESET}"
        echo -e "${WHITE}Choose an option: ${RESET}\c"
        read choice

        case "${choice,,}" in
            q) _stack_query ;;
            p) select_prefix ;;
            a) configure_autoexport ;;
            h) configure_history ;;
            f) configure_preferences ;;
            g) configure_persistence ;;
            s) save_config ;;
            x) done_configuring=true ;;
            e)
               log_conversation "System" "Exiting script from main menu."
               echo -e "${WHITE}Exiting script.${RESET}"
               exit 0
               ;;
            *) echo -e "${RED}Invalid option.${RESET}" ;;
        esac
    done
}

# --- Other functions ---
select_prefix() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Setting up prefixes..."

    mkdir -p "${PREFIXES_DIR}" || { echo -e "${RED}ERROR: Could not create prefixes directory: '${PREFIXES_DIR}'. Check permissions.${RESET}" | tee /dev/stderr; exit 1; }

    if [ ! -f "${DEFAULT_PREFIX_FILE}" ]; then
        echo "You are a helpful and concise AI assistant." > "${DEFAULT_PREFIX_FILE}"
        log_conversation "System" "Created default prefix file: '${DEFAULT_PREFIX_FILE}'."
    fi

    if [ ! -f "${TEXT_PREFIX_FILE}" ]; then
        echo "Provide a concise response in plain text that" > "${TEXT_PREFIX_FILE}"
        log_conversation "System" "Created text prefix file: '${TEXT_PREFIX_FILE}'."
    fi

    local choice_made=false
    while ! "$choice_made"; do
        local prefix_files=($(find "${PREFIXES_DIR}" -maxdepth 1 -type f -name "*.txt" -printf "%f\n" | sort))
        local num_files=${#prefix_files[@]}

        echo -e "${CYAN}=== Select a Prefix File ===${RESET}"
        echo -e "${NEON_PINK}0) No prefix (default conversation style)${RESET}"
        for i in "${!prefix_files[@]}"; do
            local filename="${prefix_files[$i]}"
            local content_preview=$(head -n 1 "${PREFIXES_DIR}/${filename}" | cut -c 1-80)
            if [ ${#content_preview} -ge 80 ]; then
                content_preview="${content_preview}..."
            fi
            echo -e "${NEON_PINK}$((i+1))) ${filename}:${RESET} ${content_preview}"
        done
        echo -e "${CYAN}===========================${RESET}"
        echo -e "${NEON_PINK}M) Make a new prefix file${RESET}"
        echo -e "${NEON_PINK}D) Delete a prefix file${RESET}"
        echo -e "${NEON_PINK}B) Back to main menu${RESET}"
        echo -e "${CYAN}===========================${RESET}"

        echo -e "${WHITE}Enter number (0-${num_files}) or option (M/D/B): ${RESET}\c"
        read choice

        case "${choice,,}" in
            m) _make_prefix_file ;;
            d) _delete_prefix_file ;;
            b) choice_made=true; return ;;
            *)
                if [[ "$choice" =~ ^[0-9]+$ ]] && (( choice >= 0 && choice <= num_files )); then
                    if [ "$choice" -eq 0 ]; then
                        SELECTED_PREFIX_FILENAME=""
                        SELECTED_PREFIX_CONTENT=""
                        AUTOEXPORT_ENABLED="false"
                        AUTOEXPORT_FILE_EXT=""
                        AUTOEXPORT_FORMAT_FUNC=""
                        log_conversation "System" "No prefix selected. Autoexport disabled."
                        echo -e "${WHITE}No prefix selected. Autoexport disabled.${RESET}"
                    else
                        local selected_filename="${prefix_files[$((choice-1))]}"
                        SELECTED_PREFIX_FILENAME="${selected_filename}"
                        SELECTED_PREFIX_CONTENT=$(cat "${PREFIXES_DIR}/${selected_filename}")
                        log_conversation "System" "Selected prefix from '${SELECTED_PREFIX_FILENAME}'."
                        echo -e "${WHITE}Prefix content: ${NEON_GREEN}\"$(head -n 1 "${PREFIXES_DIR}/${SELECTED_PREFIX_FILENAME}" | cut -c 1-80)...\"${RESET}"

                        if [ "${SELECTED_PREFIX_FILENAME}" = "powershellcript-prefix.txt" ]; then
                            AUTOEXPORT_ENABLED="true"
                            AUTOEXPORT_FILE_EXT="ps1"
                            AUTOEXPORT_FORMAT_FUNC="_format_txt_response"
                            log_conversation "System" "Detected 'powershellcript-prefix.txt'. Autoexport set to .ps1."
                            echo -e "${WHITE}Autoexport automatically set to '${NEON_GREEN}.ps1${RESET}' for PowerShell scripts."
                        elif [ "${SELECTED_PREFIX_FILENAME}" = "html-prefix.txt" ]; then
                            AUTOEXPORT_ENABLED="true"
                            AUTOEXPORT_FILE_EXT="html"
                            AUTOEXPORT_FORMAT_FUNC="_format_html_response"
                            log_conversation "System" "Detected 'html-prefix.txt'. Autoexport set to .html."
                            echo -e "${WHITE}Autoexport automatically set to '${NEON_GREEN}.html${RESET}' for HTML webpages."
                        elif [ "${SELECTED_PREFIX_FILENAME}" = "json-prefix.txt" ]; then
                            AUTOEXPORT_ENABLED="true"
                            AUTOEXPORT_FILE_EXT="json"
                            AUTOEXPORT_FORMAT_FUNC="_format_json_response"
                            log_conversation "System" "Detected 'json-prefix.txt'. Autoexport set to .json."
                            echo -e "${WHITE}Autoexport automatically set to '${NEON_GREEN}.json${RESET}' for JSON data."
                        elif [ "${SELECTED_PREFIX_FILENAME}" = "text-prefix.txt" ]; then
                            AUTOEXPORT_ENABLED="true"
                            AUTOEXPORT_FILE_EXT="txt"
                            AUTOEXPORT_FORMAT_FUNC="_format_txt_response"
                            log_conversation "System" "Detected 'text-prefix.txt'. Autoexport set to .txt."
                            echo -e "${WHITE}Autoexport automatically set to '${NEON_GREEN}.txt${RESET}' for plain text."
                        fi
                    fi
                    choice_made=true
                else
                    echo -e "${RED}Invalid choice. Please enter a number (0-${num_files}) or M/D/B.${RESET}"
                fi
                ;;
        esac
        if ! "$choice_made"; then
            clear
            echo -e "\n\n"
            log_conversation "System" "Prefix menu refreshed."
        fi
    done
}

configure_autoexport() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Configuring Autoexport feature..."

    echo -e "${CYAN}=== Autoexport Configuration ===${RESET}"
    local choice
    while true; do
        echo -e "${WHITE}Enable Autoexport? (yes/no): ${RESET}\c"
        read choice
        case "${choice,,}" in
            yes|y)
                AUTOEXPORT_ENABLED="true"
                break
                ;;
            no|n)
                AUTOEXPORT_ENABLED="false"
                AUTOEXPORT_FILE_EXT=""
                AUTOEXPORT_FORMAT_FUNC=""
                echo -e "${WHITE}Autoexport disabled.${RESET}"
                log_conversation "System" "Autoexport disabled."
                return
                ;;
            *)
                echo -e "${RED}Invalid choice. Please enter 'yes' or 'no'.${RESET}"
                ;;
        esac
    done

    if [ "${AUTOEXPORT_ENABLED}" = "true" ]; then
        echo -e "\n${CYAN}=== Select Export File Type ===${RESET}"
        echo -e "${NEON_PINK}1) Plain Text (.txt)${RESET}"
        echo -e "${NEON_PINK}2) Basic HTML (.html)${RESET}"
        echo -e "${NEON_PINK}3) PowerShell Script (.ps1)${RESET}"
        echo -e "${NEON_PINK}4) JSON (.json)${RESET}"
        echo -e "${CYAN}===============================${RESET}"
        local format_choice
        while true; do
            echo -e "${WHITE}Enter number (1-4): ${RESET}\c"
            read format_choice
            case "$format_choice" in
                1) AUTOEXPORT_FILE_EXT="txt"; AUTOEXPORT_FORMAT_FUNC="_format_txt_response"; break ;;
                2) AUTOEXPORT_FILE_EXT="html"; AUTOEXPORT_FORMAT_FUNC="_format_html_response"; break ;;
                3) AUTOEXPORT_FILE_EXT="ps1"; AUTOEXPORT_FORMAT_FUNC="_format_txt_response"; break ;;
                4) AUTOEXPORT_FILE_EXT="json"; AUTOEXPORT_FORMAT_FUNC="_format_json_response"; break ;;
                *) echo -e "${RED}Invalid choice. Please enter 1, 2, 3, or 4.${RESET}" ;;
            esac
        done
        log_conversation "System" "Autoexport enabled. File type: '.${AUTOEXPORT_FILE_EXT}', Format: '${AUTOEXPORT_FORMAT_FUNC}'."
        echo -e "${WHITE}Autoexport enabled. File type: '${NEON_GREEN}${AUTOEXPORT_FILE_EXT}${RESET}'."
    fi
}

configure_history() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Toggling Conversation History feature..."

    echo -e "${CYAN}=== Conversation History Configuration ===${RESET}"
    if [ "${ENABLE_HISTORY}" = "true" ]; then
        ENABLE_HISTORY="false"
        echo -e "${WHITE}Conversation History is now: ${NEON_GREEN}DISABLED${RESET}."
        log_conversation "System" "Conversation History disabled."
    else
        ENABLE_HISTORY="true"
        echo -e "${WHITE}Conversation History is now: ${NEON_GREEN}ENABLED${RESET}."
        log_conversation "System" "Conversation History enabled."
    fi
    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

configure_preferences() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Toggling Preferences File feature..."

    if [ ! -f "${PREF_FILE}" ]; then
        echo "You are a helpful and harmless AI. Your responses should be concise unless more detail is explicitly requested." > "${PREF_FILE}"
        log_conversation "System" "Created default preferences file: '${PREF_FILE}'."
    fi

    echo -e "${CYAN}=== Preferences File Configuration ===${RESET}"
    if [ "${ENABLE_PREFERENCES}" = "true" ]; then
        ENABLE_PREFERENCES="false"
        echo -e "${WHITE}Preferences File is now: ${NEON_GREEN}DISABLED${RESET}."
        log_conversation "System" "Preferences File disabled."
        PREFERENCES_TEXT=""
    else
        ENABLE_PREFERENCES="true"
        PREFERENCES_TEXT=$(cat "${PREF_FILE}")
        echo -e "${WHITE}Preferences File is now: ${NEON_GREEN}ENABLED${RESET}."
        echo -e "${WHITE}Content preview: ${NEON_GREEN}\"$(head -n 1 <<< "$PREFERENCES_TEXT" | cut -c 1-80)...\"${RESET}"
        log_conversation "System" "Preferences File enabled."
    fi
    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

_make_prefix_file() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Creating new prefix file..."

    local new_filename
    while true; do
        echo -e "${WHITE}Enter new prefix filename (e.g., 'my_script_prefix', will add .txt): ${RESET}\c"
        read new_filename
        if [[ -z "$new_filename" ]]; then
            echo -e "${RED}Filename cannot be empty.${RESET}"
            continue
        fi
        new_filename="${new_filename}.txt"
        if [ -f "${PREFIXES_DIR}/${new_filename}" ]; then
            echo -e "${RED}File '${new_filename}' already exists. Please choose a different name.${RESET}"
        else
            break
        fi
    done

    echo -e "${WHITE}Enter the prefix content. Press Enter, then type 'END_PREFIX' on a new line, then Enter again to finish.${RESET}"
    cat > "${PREFIXES_DIR}/${new_filename}"
    sed -i '/^END_PREFIX$/d' "${PREFIXES_DIR}/${new_filename}"

    if [ $? -eq 0 ]; then
        log_conversation "System" "Prefix file '${new_filename}' created."
        echo -e "${WHITE}Successfully created '${NEON_GREEN}${new_filename}${RESET}'."
    else
        log_conversation "ERROR" "Failed to create prefix file '${new_filename}'."
        echo -e "${RED}Failed to create '${new_filename}'.${RESET}"
    fi
    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

_delete_prefix_file() {
    clear
    echo -e "\n\n"
    log_conversation "System" "Deleting prefix file..."

    local prefix_files=($(find "${PREFIXES_DIR}" -maxdepth 1 -type f -name "*.txt" -printf "%f\n" | sort))
    local num_files=${#prefix_files[@]}

    if [ ${num_files} -eq 0 ]; then
        echo -e "${WHITE}No prefix files to delete.${RESET}"
        read -p "Press Enter to continue..."
        return
    fi

    echo -e "${CYAN}=== Select Prefix to Delete ===${RESET}"
    for i in "${!prefix_files[@]}"; do
        local filename="${prefix_files[$i]}"
        echo -e "${NEON_PINK}$((i+1))) ${filename}${RESET}"
    done
    echo -e "${CYAN}===============================${RESET}"

    local choice_to_delete
    while true; do
        echo -e "${WHITE}Enter number to delete (1-${num_files}, or 0 to cancel): ${RESET}\c"
        read choice_to_delete
        if [[ "$choice_to_delete" =~ ^[0-9]+$ ]] && (( choice_to_delete >= 0 && choice_to_delete <= num_files )); then
            break
        else
            echo -e "${RED}Invalid choice. Please enter a number from 0 to ${num_files}.${RESET}"
        fi
    done

    if [ "$choice_to_delete" -eq 0 ]; then
        echo -e "${WHITE}Deletion cancelled.${RESET}"
        read -p "Press Enter to continue..."
        return
    fi

    local file_to_delete="${prefix_files[$((choice_to_delete-1))]}"

    if [ "${file_to_delete}" = "default.txt" ]; then
        echo -e "${RED}ERROR: 'default.txt' cannot be deleted. It is a core file.${RESET}"
        read -p "Press Enter to continue..."
        return
    fi

    local confirm
    echo -e "${WHITE}Are you sure you want to delete '${NEON_GREEN}${file_to_delete}${RESET}'? (y/N): ${RESET}\c"
    read confirm
    if [[ "${confirm,,}" == "y" ]]; then
        rm "${PREFIXES_DIR}/${file_to_delete}"
        if [ $? -eq 0 ]; then
            log_conversation "System" "Prefix file '${file_to_delete}' deleted."
            echo -e "${WHITE}Successfully deleted '${NEON_GREEN}${file_to_delete}${RESET}'."
        else
            log_conversation "ERROR" "Failed to delete prefix file '${file_to_delete}'."
            echo -e "${RED}Failed to delete '${file_to_delete}'.${RESET}"
        fi
    else
        echo -e "${WHITE}Deletion cancelled.${RESET}"
    fi
    echo -e "${WHITE}Press Enter to continue...${RESET}"
    read
}

# --- Main Script Logic ---
clear
log_conversation "System" "Starting begem.sh conversation."

if [ ! -d "${BEGEM_HISTORY_DIR}" ]; then
    mkdir -p "${BEGEM_HISTORY_DIR}"
    if [ $? -ne 0 ]; then
        echo -e "${RED}ERROR: Failed to create history directory: '${BEGEM_HISTORY_DIR}'. Check permissions.${RESET}" | tee /dev/stderr
        exit 1
    fi
    log_conversation "System" "Created history directory: '${BEGEM_HISTORY_DIR}'."
fi
if [ ! -d "${STACK_DIR}" ]; then
    mkdir -p "${STACK_DIR}"
    if [ $? -ne 0 ]; then
        echo -e "${RED}ERROR: Failed to create stack directory: '${STACK_DIR}'. Check permissions.${RESET}" | tee /dev/stderr
        exit 1
    fi
    log_conversation "System" "Created stack directory: '${STACK_DIR}'."
fi
if [ ! -d "${PERSIST_DIR}" ]; then
    mkdir -p "${PERSIST_DIR}"
    if [ $? -ne 0 ]; then
        echo -e "${RED}ERROR: Failed to create persistence directory: '${PERSIST_DIR}'. Check permissions.${RESET}" | tee /dev/stderr
        exit 1
    fi
    log_conversation "System" "Created persistence directory: '${PERSIST_DIR}'."
fi

read_api_key

if [ -f "${TEXT_PREFIX_FILE}" ]; then
    SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
else
    echo "Provide a concise response in plain text that" > "${TEXT_PREFIX_FILE}"
    SELECTED_PREFIX_CONTENT=$(cat "${TEXT_PREFIX_FILE}")
    log_conversation "System" "Created and loaded text prefix file: '${TEXT_PREFIX_FILE}'."
fi

if load_config; then
    : # No-op, just proceed
else
    show_main_config_menu
fi

if [ "${ENABLE_HISTORY}" = "true" ]; then
    current_date=$(date +%Y-%m-%d)
    daily_history_dir="${HISTORY_BASE_DIR}/${current_date}"
    CURRENT_SESSION_HISTORY_FILE="${daily_history_dir}/history.txt"

    mkdir -p "${daily_history_dir}" || { log_conversation "ERROR" "Failed to create daily history directory: '${daily_history_dir}'. Check permissions."; echo -e "${RED}Failed to create daily history directory.${RESET}"; }

    if [ -d "${daily_history_dir}" ] && [ ! -f "${CURRENT_SESSION_HISTORY_FILE}" ]; then
        touch "${CURRENT_SESSION_HISTORY_FILE}"
        if [ $? -eq 0 ]; then
            log_conversation "System" "New conversation history file created: '${CURRENT_SESSION_HISTORY_FILE}'."
        else
            log_conversation "ERROR" "Failed to create conversation history file: '${CURRENT_SESSION_HISTORY_FILE}'."
            echo -e "${RED}Failed to create conversation history file.${RESET}"
        fi
    fi
fi

log_conversation "System" "Gemini API key loaded. Type 'exit' to quit."
echo -e "${WHITE}Gemini: Hello! How can I help you today?${RESET}"

while true; do
    echo -e "${WHITE}You: ${RESET}\c"
    read user_query_input

    if [ -z "$user_query_input" ]; then
        continue
    fi

    if [ "$user_query_input" = "exit" ]; then
        log_conversation "System" "Exiting conversation."
        save_config
        break
    fi

    log_conversation "You" "${user_query_input}"
    if [ "${ENABLE_HISTORY}" = "true" ] && [ -n "${CURRENT_SESSION_HISTORY_FILE}" ]; then
        echo "You: ${user_query_input}" >> "${CURRENT_SESSION_HISTORY_FILE}"
    fi

    local query_to_send="${user_query_input}"
    local persistence_prefix="${PERSISTENCE_PREFIX}"
    if [ "${ENABLE_PREFERENCES}" = "true" ] && [ -n "${PREFERENCES_TEXT}" ]; then
        persistence_prefix="${PREFERENCES_TEXT}"
    fi
    if [ "${ENABLE_PERSISTENCE}" = "true" ]; then
        local transcript_content=""
        [ -f "${PERSIST_TRANSCRIPT_FILE}" ] && transcript_content=$(cat "${PERSIST_TRANSCRIPT_FILE}")
        query_to_send="[*[*[*!! historical transcription !!! *]*]*]*\n${transcript_content}\n[*[*[*!! end of historical transcription !!! *]*]*]*\n\n${persistence_prefix}\n${user_query_input}"
    fi

    gemini_response=$(query_gemini "${query_to_send}" "${GEMINI_MODEL}")

    if [ $? -eq 0 ]; then
        log_conversation "Gemini" "${gemini_response}"
        if [ "${ENABLE_HISTORY}" = "true" ] && [ -n "${CURRENT_SESSION_HISTORY_FILE}" ]; then
            echo "Gemini: ${gemini_response}" >> "${CURRENT_SESSION_HISTORY_FILE}"
        fi
        echo -e "${WHITE}Gemini: ${RESET}${gemini_response}"

        if [ "${AUTOEXPORT_ENABLED}" = "true" ]; then
            log_conversation "System" "Autoexport is enabled. Settings: EXT=${AUTOEXPORT_FILE_EXT}, FUNC=${AUTOEXPORT_FORMAT_FUNC}"
            current_date=$(date +%Y-%m-%d)
            daily_export_dir="${EXPORTS_BASE_DIR}/${current_date}"

            mkdir -p "${daily_export_dir}" || { log_conversation "ERROR" "Failed to create daily export directory: '${daily_export_dir}'. Check permissions."; echo -e "${RED}Failed to create daily export directory.${RESET}"; continue; }

            if [ -d "${daily_export_dir}" ]; then
                export_filename="response_$(date +%H%M%S)_$(( RANDOM % 1000 )).${AUTOEXPORT_FILE_EXT}"
                export_filepath="${daily_export_dir}/${export_filename}"
                log_conversation "System" "Attempting to export to '${export_filepath}'"

                if [ -z "${export_filepath}" ]; then
                    log_conversation "ERROR" "Export filepath is empty. Skipping export."
                    continue
                fi

                if [ -z "${AUTOEXPORT_FILE_EXT}" ]; then
                    log_conversation "ERROR" "AUTOEXPORT_FILE_EXT is empty. Skipping export."
                    continue
                fi

                if type -t "${AUTOEXPORT_FORMAT_FUNC}" &>/dev/null && [ "$(type -t "${AUTOEXPORT_FORMAT_FUNC}")" = "function" ]; then
                    formatted_response=$("${AUTOEXPORT_FORMAT_FUNC}" "${gemini_response}")
                else
                    log_conversation "ERROR" "Formatter function '${AUTOEXPORT_FORMAT_FUNC}' not found. Exporting raw text."
                    formatted_response="${gemini_response}"
                fi

                if [ -z "${formatted_response}" ]; then
                    log_conversation "ERROR" "Formatted response is empty. Skipping export to '${export_filepath}'."
                    echo -e "${RED}Error: Formatted response is empty.${RESET}"
                    continue
                fi

                echo "${formatted_response}" > "${export_filepath}"
                if [ $? -eq 0 ]; then
                    log_conversation "System" "Gemini response exported to '${export_filepath}'."
                    echo -e "${WHITE}Exported response to '${NEON_GREEN}${export_filepath}${RESET}'."
                else
                    log_conversation "ERROR" "Failed to export response to '${export_filepath}'. Check permissions."
                    echo -e "${RED}Failed to export response to '${export_filepath}'.${RESET}"
                fi
            fi
        else
            log_conversation "System" "Autoexport is disabled. No file will be created."
        fi
    else
        log_conversation "Gemini" "Error: Could not get response."
        echo -e "${RED}Gemini: Error getting response. See logs and terminal output for details.${RESET}"
    fi
done

log_conversation "System" "Conversation ended."
