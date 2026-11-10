Branch: chore/local-settings-template, created from main.

Goal: provide a placeholder local.settings.json template with the required keys, and make sure real values can never be committed.

Steps:
1. Check whether a local.settings.json already exists in the repo. Do not read or print its values if it contains real secrets. Report only whether it exists and whether git tracks it (git ls-files).
2. Create local.settings.example.json at the repo root (or in src/PipelineRunner if that is where the Function App settings live; report which location you chose and why) with this structure:

{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "AOAI_ENDPOINT": "https://<your-resource>.openai.azure.com/",
    "AOAI_CHAT_DEPLOYMENT": "<chat-deployment-name>",
    "AOAI_EMBED_DEPLOYMENT": "<embedding-deployment-name>",
    "SQL_CONNECTION": "Server=tcp:<server>.database.windows.net,1433;Database=<db>;Authentication=Active Directory Default;Encrypt=True;"
  }
}

   Placeholders must be obviously fake. Do not put any real endpoint, key, or connection string in this file.
3. Ensure ignore rules: add local.settings.json to .gitignore if it is not already there. Verify with git check-ignore -v local.settings.json.
4. Do not add AOAI_API_KEY to the template. Note in the report that key auth is optional and should come from the environment or Key Vault, not a file.
5. Document the keys in a short README section or in the example file's sibling README.md, with one line per key describing what it is.

Constraints:
- Do not modify any file in db/, src/Domain/, or schema/.
- Do not stage or commit the real local.settings.json if it exists.
- Do not touch the uncommitted changes to README.md, docs/, or src/ files already in the working tree. If the README needs the new section, report the change and do not write it to README.md; put it in a new file instead.

Report: whether local.settings.json exists and is tracked, the path of the example file, the output of git check-ignore -v local.settings.json, and the git status --short output.
