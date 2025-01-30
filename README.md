#!/bin/bash
#-----------------------------------------------------------------
# chmod +x migrate_repos_with_logging.sh
# ./migrate_repos_with_logging.sh repos.csv
#-----------------------------------------------------------------

# Load environment variables from the .env file
if [[ -f .env ]]; then
    echo "Loading environment variables from .env file..."
    export $(grep -v '^#' .env | xargs)
else
    echo "Error: .env file not found."
    exit 1
fi

# Check if all required environment variables are set
if [[ -z "$GH_PAT" || -z "$GH_SOURCE_PAT" || -z "$TARGET_API_URL" || -z "$GHES_API_URL" || -z "$AZURE_STORAGE_CONNECTION_STRING" ]]; then
    echo "Error: One or more required environment variables are missing in the .env file."
    echo "Please include the following variables in the .env file:"
    echo "  GH_PAT, GH_SOURCE_PAT, TARGET_API_URL, GHES_API_URL, AZURE_STORAGE_CONNECTION_STRING"
    exit 1
fi

# Check if CSV file is provided as an argument
if [[ $# -ne 1 ]]; then
    echo "Usage: $0 <csv_file>"
    echo "The CSV file should contain the following headers: SOURCE,CURRENT-NAME,DESTINATION,NEW-NAME"
    exit 1
fi

CSV_FILE="$1"
LOG_FILE="migration_log_$(date +%Y%m%d_%H%M%S).log"
SUCCESS_COUNT=0
FAILURE_COUNT=0

# Ensure no Windows-style line endings in the CSV file
sed -i '' 's/\r$//' "$CSV_FILE"

# Export required tokens (already loaded from .env)
export GH_PAT
export GH_SOURCE_PAT

# Log start time
echo "Migration process started at $(date)" | tee -a "$LOG_FILE"

echo "Reading from CSV file: $CSV_FILE" | tee -a "$LOG_FILE"

# Read CSV file and process each row while skipping the header
tail -n +2 "$CSV_FILE" | while IFS=',' read -r SOURCE CURRENT_NAME DESTINATION NEW_NAME; do
    # Skip empty lines
    [[ -z "$SOURCE" ]] && continue  

    # Sanitize NEW-NAME by removing trailing dashes
    CLEAN_NEW_NAME=$(echo "$NEW_NAME" | sed 's/-$//')

    echo "-------------------------------------------" | tee -a "$LOG_FILE"
    echo "Starting migration for Source Org: $SOURCE, Source Repo: $CURRENT_NAME" | tee -a "$LOG_FILE"
    echo "Target Org: $DESTINATION, Target Repo: $CLEAN_NEW_NAME" | tee -a "$LOG_FILE"

    # Execute the migration command
    gh gei migrate-repo \
        --github-source-org "$SOURCE" \
        --source-repo "$CURRENT_NAME" \
        --github-target-org "$DESTINATION" \
        --target-repo "$CLEAN_NEW_NAME" \
        --ghes-api-url "$GHES_API_URL" \
        --azure-storage-connection-string "$AZURE_STORAGE_CONNECTION_STRING" >> "$LOG_FILE" 2>&1

    # Check for errors
    if [[ $? -eq 0 ]]; then
        echo "✅ Migration for $CURRENT_NAME to $CLEAN_NEW_NAME completed successfully!" | tee -a "$LOG_FILE"
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
    else
        echo "❌ Migration for $CURRENT_NAME to $CLEAN_NEW_NAME failed. Check logs for details." | tee -a "$LOG_FILE"
        FAILURE_COUNT=$((FAILURE_COUNT + 1))
    fi
    echo "-------------------------------------------" | tee -a "$LOG_FILE"
done

# Print summary
echo "Migration process completed at $(date)" | tee -a "$LOG_FILE"
echo "-------------------------------------------" | tee -a "$LOG_FILE"
echo "Summary:" | tee -a "$LOG_FILE"
echo "  ✅ Successful migrations: $SUCCESS_COUNT" | tee -a "$LOG_FILE"
echo "  ❌ Failed migrations: $FAILURE_COUNT" | tee -a "$LOG_FILE"
echo "For details, check the log file: $LOG_FILE"
