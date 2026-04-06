# whatsapp-mcp — LID Fix

Patch for [whatsapp-mcp](https://github.com/lharries/whatsapp-mcp) that fixes contact lookup when WhatsApp uses LID (Linked Device ID) JIDs instead of phone number JIDs.

## The Problem

WhatsApp migrated contacts from phone-based JIDs (`15550001234@s.whatsapp.net`) to internal LID JIDs (`123456789012345@lid`) for privacy reasons. The original `whatsapp.py` only searched by phone number, so:

- `search_contacts("John")` returned nothing even if John was in your contacts
- `get_direct_chat_by_contact("1555...")` returned `None` even though the chat existed
- `list_messages(sender_phone_number="1555...")` returned an empty list
- Contact names showed as raw numeric IDs instead of real names

## The Fix

This patch adds three helpers and updates four functions in `whatsapp.py`:

### New Helpers

**`_normalize_phone(phone)`** — Strips `+`, spaces, and dashes from any phone number format.

**`_resolve_phone_to_jids(phone)`** — Given a phone number, returns all matching JIDs (regular `@s.whatsapp.net` + `@lid`) by querying the `whatsmeow_lid_map` table in `whatsapp.db`. Uses suffix matching (last 10 digits) to handle WhatsApp's varying number formats (e.g. `15550001234` vs `115550001234`).

**`_get_contact_name(phone)`** — Looks up the contact's full name or push name in `whatsmeow_contacts` table.

### Fixed Functions

| Function | Fix |
|---|---|
| `search_contacts` | Queries `whatsapp.db` first (has real names + LID contacts), falls back to `messages.db` |
| `get_direct_chat_by_contact` | Uses `_resolve_phone_to_jids` to match any JID variant; resolves name from store |
| `list_messages` | Filters by all JID variants when `sender_phone_number` is set |
| `list_messages` | Skips `include_context` expansion when filtering by phone (avoids duplicate messages) |

## Installation

Replace `whatsapp-mcp-server/whatsapp.py` in your `whatsapp-mcp` installation with the patched version:

```bash
cp whatsapp.py /path/to/whatsapp-mcp/whatsapp-mcp-server/whatsapp.py
```

Then restart your MCP server (in Claude Code VSCode: `Developer: Reload Window`).

## Verification

After patching, these should work:

```
search_contacts("John")                        # Returns contacts with real names
get_direct_chat_by_contact("+1 555 000 1234")  # Returns chat by phone number
list_messages(sender_phone_number="1555...")   # Returns clean conversation
```

## Windows MCP Setup

If you are on Windows and the WhatsApp MCP tools do not appear in Claude Code, the issue is that `cmd /c script.bat` prints a Windows banner to stdout, corrupting the MCP stdio channel.

**Fix:** Use a Python launcher instead of the batch file.

Create `~/.claude/whatsapp-launcher.py`:

```python
import subprocess, sys, os, time

BRIDGE_EXE = r"C:\path\to\whatsapp-mcp\whatsapp-bridge\whatsapp-bridge.exe"
MCP_SERVER_DIR = r"C:\path\to\whatsapp-mcp\whatsapp-mcp-server"
UV_EXE = r"C:\Users\<user>\.local\bin\uv.exe"

def main():
    env = os.environ.copy()
    env.pop("VIRTUAL_ENV", None)  # Prevent uv from recreating venv on every start
    try:
        subprocess.Popen(
            [BRIDGE_EXE],
            stdin=subprocess.DEVNULL,
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
            creationflags=subprocess.CREATE_NO_WINDOW,
            env=env,
        )
    except Exception:
        pass
    time.sleep(2)
    result = subprocess.run(
        [UV_EXE, "run", "--python", "3.12", "--directory", MCP_SERVER_DIR, "main.py"],
        stdin=sys.stdin, stdout=sys.stdout, stderr=sys.stderr, env=env,
    )
    sys.exit(result.returncode)

if __name__ == "__main__":
    main()
```

Then register it in Claude Code:

```bash
claude mcp add --scope user whatsapp "C:\Users\<user>\miniconda3\python.exe" "C:\Users\<user>\.claude\whatsapp-launcher.py"
```

Reload the VSCode window (`Developer: Reload Window`) and the WhatsApp tools will appear.

## Root Cause Reference

- `whatsapp.db` — WhatsApp credential/contact store (Go bridge writes this)
- `whatsmeow_lid_map` table — maps `lid` (internal ID) to `pn` (phone number)
- `whatsmeow_contacts` table — stores `full_name`, `push_name` indexed by `their_jid`
- `messages.db` — message history (Python MCP server reads this)

The original code only used `messages.db` for contact lookup, which does not have real names for LID contacts.
