# dotenv: read and edit .env files from bash

```shell
__dotenv=
__dotenv_file=
__dotenv_cmd=.env

.env() {
	REPLY=()
	[[ $__dotenv_file || ${1-} == -* ]] || .env.--file .env || return
	if declare -F -- ".env.${1-}" >/dev/null; then .env."$@"; return ; fi
	.env --help >&2; return 64
}

.env.-f() { .env.--file "$@"; }

.env.get() {
	.env::arg "get requires a key" "$@" &&
	[[ "$__dotenv" =~ ^(.*(^|$'\n'))([ ]*)"$1="(.*)$ ]] &&
	REPLY=${BASH_REMATCH[4]%%$'\n'*} && REPLY=${REPLY%"${REPLY##*[![:space:]]}"}
}

.env.parse() {
	local line key
	while IFS= read -r line; do
		line=${line#"${line%%[![:space:]]*}"}  # trim leading whitespace
		line=${line%"${line##*[![:space:]]}"}  # trim trailing whitespace
		if [[ ! "$line" || "$line" == '#'* ]]; then continue ; fi
		if (($#)); then
			for key; do
				if [[ $key == "${line%%=*}" ]]; then REPLY+=("$line"); break;
				fi
			done
		else
			REPLY+=("$line")
		fi
	done <<<"$__dotenv"
	((${#REPLY[@]}))
}

.env.export() {	! .env.parse "$@" || export "${REPLY[@]}"; }

.env.set() {
	.env::file load || return ; local key saved=$__dotenv
	while (($#)); do
		key=${1#+}; key=${key%%=*}
		if .env.get "$key"; then
			REPLY=()
			if [[ $1 == +* ]]; then shift; continue  # skip if already found
			elif [[ $1 == *=* ]]; then
				__dotenv=${BASH_REMATCH[1]}${BASH_REMATCH[3]}$1$'\n'${BASH_REMATCH[4]#*$'\n'}
			else
				__dotenv=${BASH_REMATCH[1]}${BASH_REMATCH[4]#*$'\n'}
				continue   # delete all occurrences
			fi
		elif [[ $1 == *=* ]]; then
			__dotenv+="${1#+}"$'\n'
		fi
		shift
	done
	[[ $__dotenv == "$saved" ]] || .env::file save
}

.env.puts() { echo "${1-}">>"$__dotenv_file" && __dotenv+="$1"$'\n'; }

.env.generate() {
	.env::arg "key required for generate" "$@" || return
	.env.get "$1" && return || REPLY=$("${@:2}") || return
	.env::one "generate: ouptut of '${*:2}' has more than one line" "$REPLY" || return
	.env.puts "$1=$REPLY"
}

.env.--file() {
	.env::arg "filename required for --file" "$@" || return
	__dotenv_file=$1; .env::file load || return
	(($#<2)) || .env "${@:2}"
}

.env::arg() { [[ "${2-}" ]] || { echo "$__dotenv_cmd: $1" >&2; return 64; }; }

.env::one() { [[ "$2" != *$'\n'* ]] || .env::arg "$1"; }

.env::file() {
	local REPLY=$__dotenv_file
	case "$1" in
	load)
		__dotenv=; ! [[ -f "$REPLY" ]] || __dotenv="$(<"$REPLY")"$'\n' || return ;;
	save)
		if [[ -L "$REPLY" ]] && declare -F -- realpath.resolved >/dev/null; then
			realpath.resolved "$REPLY"
		fi
		{ [[ ! -f "$REPLY" ]] || cp -p "$REPLY" "$REPLY.bak"; } &&
		printf %s "$__dotenv" >"$REPLY.bak" && cp -f "$REPLY.bak" "$REPLY" && rm -f "$REPLY.bak"
	esac
}
```
