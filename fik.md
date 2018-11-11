# `fik`: TOML files for traefik routes

```shell @mdsh
@module fik.md
@main loco_main

@require pjeby/license @comment    "LICENSE"
@require bashup/jqmd   mdsh-source "$BASHER_PACKAGES_PATH/bashup/jqmd/jqmd.md"
@require bashup/loco   mdsh-source "$BASHER_PACKAGES_PATH/bashup/loco/loco.md"
@require bashup/dotenv mdsh-embed  "$BASHER_PACKAGES_PATH/bashup/dotenv/dotenv"
@require bashup/events tail -n +2  "$BASHER_PACKAGES_PATH/bashup/events/bashup.events"
@require .devkit/tty   tail -n +2  "$DEVKIT_HOME/modules/tty"

echo tty_prefix=FIK_   # Use FIK_ISATTY, FIK_PAGER, etc.
```

## Configuration

### Config Files

```shell
loco_preconfig() {
    LOCO_FILE=(fik.json fik.yml .fik fik.md)
    LOCO_NAME=fik
    LOCO_USER_CONFIG=$HOME/.config/fikrc
    LOCO_SITE_CONFIG=/etc/traefik/fikrc

    FIK_ROUTES=/etc/traefik/routes
    FIK_TOML=
    FIK_PROJECT=
}

uuid4() { pypipe "str(uuid.uuid4())" "uuid" </dev/null; }
as-routefile() { event encode "$1"; realpath.absolute "$FIK_ROUTES" "fik-$REPLY.toml"; }
project-key() { FIK_PROJECT="$1"; as-routefile "$1"; FIK_TOML=$REPLY; }

loco_loadproject() {
	cd "$LOCO_ROOT"
	.env -f .fik-project export
	for REPLY in "${LOCO_FILE[@]}"; do
		[[ -f "$REPLY" ]] || continue
		case $REPLY in
		fik.json) JSON "$(<fik.json)" ;;
		fik.yml)  YAML "$(<fik.yml)"  ;;
		.fik)     source ./.fik ;;
		fik.md)   mdsh-run fik.md ;;
		esac
	done
	if [[ ! ${FIK_TOML-} ]]; then
		if [[ ! ${FIK_PROJECT-} ]]; then
			.env -f .fik-project generate FIK_PROJECT uuid4
			.env export FIK_PROJECT
		fi
		project-key "$FIK_PROJECT"
	fi
	event emit project-loaded
}

mdsh-error() { printf -v REPLY "$1"'\n' "${@:2}"; loco_error "$REPLY"; }
unset -f mdsh:file-footer  # don't add jqmd footer to compiled .md files
```

### Routing Rules

```shell
declare -gA fik_rule_counts=() fik_url_counts=()

frontend-set() { APPLY ".frontends[\$fik_frontend] |= ($1)" "${@:2}"; }
backend-set() { APPLY ".backends[\$fik_backend] |= ($1)" "${@:2}"; }

backend() {
	fik_backend=$1; APPLY . fik_backend
	event emit backend "$@";
	while shift; (($#)); do server "$1"; done
}

match() {
	fik_frontend=$1; APPLY . fik_frontend
	frontend-set '.backend=$fik_backend'
	event emit frontend "$@";
	while shift; (($#)); do must-have "$1"; shift; done
}

priority() { frontend-set ".priority=$1"; }
pass-host() { frontend-set '.passHostHeader=$bool' @bool="${1-true}"; }

must-have() { __series frontend routes  rule "$1"; }
server()    { __series backend  servers url  "$1"; } # "$fik_backend"; }

__series() {
	local -n key=fik_${1} map=fik_${3}_counts; local itemno=$((++map["$key"]))
	printf -v itemno %s%03d "${5-$3}" "$itemno"
	APPLY ".$1s[\$fik_$1].$2[\$itemno].$3 = \$val" itemno val="$4"
	event emit "$3" "$itemno" "$4"
}

tls() {
	FILTER '.tls+=[{certificate: {certFile:%s, keyFile:%s}}]' "$1" "$2"
	if (($#>2)); then
		JSON-LIST "${@:3}"; FILTER ".tls[-1].entryPoints=$REPLY"
	fi
}

redirect() {
	frontend-set '.redirect |= ( .regex = $from | .replacement = $to | .permanent = $perm )' \
		from="$1" to="$2" @perm="${3:-false}"
}

require-ssl() {
	frontend-set '.headers |= ( .SSLRedirect = $redir | .SSLTemporaryRedirect = $temp )' \
		@redir="${1-true}" @temp="${2-${1:-false}}"
}
```

### Unique Backends

```shell
DEFINE '
	def fik::unique_backend($frontend): .
		| .frontends[$frontend].backend as $original
		| "\($original): \($frontend)" as $unique_name
		| .backends[$unique_name] = .backends[$original]
		| .frontends[$frontend].backend = $unique_name
	;
'
unique-backends() { event "${1:-on}" frontend unique-backend; }

unique-backend() {
	event on project-loaded FILTER 'fik::unique_backend(%s)' "${1-$fik_frontend}"
}
```

## Commands

```shell
fik.filename() { echo "$FIK_TOML"; }
fik.key()      { echo "$FIK_PROJECT"; }

fik.up()    {
	mkdir -p "$FIK_ROUTES"
	fik toml >"$FIK_TOML".tmp
	mv "$FIK_TOML".tmp "$FIK_TOML"
}

fik.down()  { [[ ! -f "$FIK_TOML" ]] || rm "$FIK_TOML"; }

fik.diff()  { tty pager highlight-diff diffcolor -- diff -u <(fik old-json) <(fik json); }
fik.watch() { watch-continuous 2 fik diff; }

fik.old-json() {
	[[ -f "$FIK_TOML" ]] || local FIK_TOML=/dev/null
	toml2json <"$FIK_TOML" | jq-tty -S .
}

```

### Routefile Generation

```shell
fik.toml()     { fik json -c | json2toml | colorize toml; }
fik.json()     { local JQ_CMD=(jq-tty); RUN_JQ -n -S "$@"; }

json2toml() { pypipe "pytoml.dumps(json.loads(s), sort_keys=True)" "pytoml,json"; }
toml2json() { pypipe "json.dumps(pytoml.loads(s))" "pytoml,json"; }

pypipe() {
	python -c "import sys${2:+,$2}; sys.stdout.write((lambda s: $1)(sys.stdin.read()))"
}

```

### Paging, Coloring, and Watching

```shell
require-any() { :; }  # XXX

colorize() { tty pager colorize-"$1" -- "${2-cat}" "${@:3}"; }
highlight-diff() { tty-tool DIFF_HIGHLIGHT diff-highlight; }

watch-continuous() {
	local interval=$1 oldint oldwinch; oldint=$(trap -p SIGINT); oldwinch="$(trap -p SIGWINCH)"
	shift; trap "continue" SIGWINCH; trap "break" SIGINT
	while :; do watch-once "$@"; sleep "$interval" & wait $! || true; done
	${oldwinch:-trap -- SIGWINCH}; ${oldint:-trap -- SIGINT}
}

watch-once() {
	local cols; cols=$(tput cols) 2>/dev/null || cols=80
	REPLY="Every ${interval}s: $*"
	clear; printf '%s%*s\n\n' "$REPLY" $((cols-${#REPLY})) "$(date "+%Y-%m-%d %H:%M:%S")"
	local -x ${tty_prefix-DEVKIT_}PAGER="pager.screenfull 3"
	"$@" || true
}
```

