I have long referred to [The Sharat's Shell Script Best Practices](https://sharats.me/posts/shell-script-best-practices/) as a starting point for my scripts, but I was working on an email processing script and ended up making a few tweaks I thought constituted good defaults. That article is still great explanation of the motivation behind this template, which I won't repeat. Here's my new template in case it's useful to anyone else:

```bash
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

USAGE="Usage: $0 [options] PARAM1

DESCRIPTION
    This program does something!

OPTIONS
    --opt1 (type)
    Description."

if [[ "${TRACE-0}" == "1" ]]; then
    set -o xtrace
fi

if [[ "${1-}" =~ ^-*h(elp)?$ ]]; then
    echo "$USAGE"
    exit
fi

cleanup() {
}

trap cleanup EXIT

main() {
    local PARAM1=""
    local OPT1=""

    while [[ $# -gt 0 ]]; do
        case "$1" in
            --opt1)
                shift
                if [[ $# -eq 0 ]]; then
                    echo "Error: --opt1 requires an argument."
                    echo "$USAGE"
                    exit 1
                fi
                OPT1="$1"
                ;;
            -*)
                echo "Unknown option: $1"
                echo "$USAGE"
                exit 1
                ;;
            *)
                PARAM1="$1"
                ;;
        esac
        shift
    done

    if [ -z "$PARAM1" ]; then
        echo "Error: PARAM1 is a required parameter."
        echo "$USAGE"
        exit 1
    fi

    if [ -z "$OPT1" ]; then
        echo "Error: --opt1 is a required option."
        echo "$USAGE"
        exit 1
    fi

	# code here
}

main "$@"
```
