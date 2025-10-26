
---

# 4 — Log Normalizer & Timeline Builder (Node CLI)  
**What:** Ingests syslog-like files, normalizes entries into JSONL, anonymizes IPs, and exports a timeline CSV.

**File:** `src/log_timeline.js`

```javascript
#!/usr/bin/env node
/**
 * Log Normalizer & Timeline Builder
 * Usage: node src/log_timeline.js /path/to/syslog
 *
 * Parses simple syslog lines, normalizes, anonymizes IP addresses, and writes timeline_export.csv
 */

const fs = require('fs');
const path = require('path');

function parseSyslogLine(line){
  // naive parser: "<Mmm dd hh:mm:ss> host proc: message"
  const m = line.match(/^([A-Z][a-z]{2}\s+\d+\s+\d{2}:\d{2}:\d{2})\s+(\S+)\s+(\S+):\s+(.*)$/);
  if (!m) return null;
  return { ts: m[1], host: m[2], proc: m[3], message: m[4] };
}

function anonymize(msg){
  // simple IP mask
  return msg.replace(/\b\d{1,3}(?:\.\d{1,3}){3}\b/g, '[IP]');
}

function processFile(p){
  const out = [];
  const lines = fs.readFileSync(p, 'utf8').split(/\r?\n/);
  for (const l of lines) {
    if (!l.trim()) continue;
    const rec = parseSyslogLine(l);
    if (rec) {
      rec.message = anonymize(rec.message);
      out.push(rec);
    }
  }
  return out;
}

if (require.main === module){
  const argv = process.argv.slice(2);
  if (argv.length === 0) { console.log('Usage: node src/log_timeline.js file1 [file2 ...]'); process.exit(1); }
  const all = [];
  for (const f of argv){
    if (!fs.existsSync(f)) { console.warn('Missing', f); continue; }
    all.push(...processFile(f));
  }
  // sort by ts string (best-effort)
  all.sort((a,b)=> a.ts.localeCompare(b.ts));
  // write CSV
  const out = ['timestamp,host,proc,message'];
  for (const r of all) out.push(`"${r.ts}","${r.host}","${r.proc}","${r.message.replace(/"/g,'""')}"`);
  fs.writeFileSync('timeline_export.csv', out.join('\n'), 'utf8');
  console.log('Wrote timeline_export.csv with', all.length, 'records');
}
