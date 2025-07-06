The `~/.bashrc` file is a script that Bash executes every time a new interactive non-login shell session starts. This makes it the ideal place for personal shell customizations like these history settings.

**Steps:**

1.  **Open your `~/.bashrc` file in a text editor:**

    ```bash
    vim ~/.bashrc
    ```

2.  **Add the lines:** Scroll to the end of the file (or find a suitable section for history settings) and paste the entire block of code:

    ```bash
    # Enhanced Bash History Settings
    # Set the common history file
    export HISTFILE=~/.common_bash_history
    export HISTTIMEFORMAT="%Y-%m-%d %T "
    export HISTCONTROL=ignoreboth

    # Number of commands to keep in the history list in memory
    export HISTSIZE=10000

    # Number of commands to save in the history file
    export HISTFILESIZE=20000

    # Append to the history file, don't overwrite it
    shopt -s histappend

    # After each command, append to the history file and reread it
    export PROMPT_COMMAND="${PROMPT_COMMAND:+$PROMPT_COMMAND$'\n'}history -a; history -c; history -r"
    ```

3.  **Save and Close:**

      * :wq

4.  **Apply Changes (for current session):**
    To make these changes effective immediately in your current terminal session without closing and reopening it, source your `~/.bashrc` file:

    ```bash
    source ~/.bashrc
    # or . ~/.bashrc
    ```
    