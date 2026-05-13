const express = require('express');
const fs = require('fs');
const path = require('path');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;
const DATA_FILE = path.join(__dirname, 'incidents.json');
const SECRET = process.env.WEBHOOK_SECRET || 'noc-secret-2024';

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// ── helpers ───────────────────────────────────────────────────────────────────

function loadData() {
  if (!fs.existsSync(DATA_FILE)) return [];
  try { return JSON.parse(fs.readFileSync(DATA_FILE, 'utf8')); }
  catch { return []; }
}

function saveData(data) {
  fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2));
}

function extractSiteIds(text) {
  const rx = /\b([A-Z]{2,5}[\s_-]?[A-Z]?\d{3,6})\b/g;
  const found = new Set();
  let m;
  while ((m = rx.exec(text)) !== null) found.add(m[1].replace(/[\s_-]/g, ''));
  return [...found];
}

// Parse time string like "12-May 22:41" into a Date (uses current year)
function parseNocTime(timeStr) {
  if (!timeStr) return null;
  try {
    const year = new Date().getFullYear();
    return new Date(`${timeStr}-${year}`);
  } catch { return null; }
}

// Calculate downtime in minutes between two NOC time strings
function calcDowntimeMinutes(startStr, endStr) {
  const start = parseNocTime(startStr);
  const end   = parseNocTime(endStr);
  if (!start || !end) return null;
  const diff = Math.round((end - start) / 60000);
  return diff > 0 ? diff : null;
}

function formatDowntime(minutes) {
  if (!minutes) return null;
  if (minutes < 60) return `${minutes}m`;
  const h = Math.floor(minutes / 60);
  const m = minutes % 60;
  return m > 0 ? `${h}h ${m}m` : `${h}h`;
}

function getSection(text, label) {
  const rx = new RegExp(label + ':\\s*([\\s\\S]*?)(?=\\n[A-Z][\\w &]+:|$)', 'i');
  const m = text.match(rx);
  return m ? m[1].trim() : '';
}

// Detect if message is a Clearance
function isClearance(text) {
  return /^clearance\s*:/i.test(text.trim());
}

function parseSMS(raw) {
  const text      = raw.trim();
  const firstLine = text.split('\n')[0];
  const parts     = firstLine.split(':').map(s => s.trim());
  const clearance = isClearance(text);

  const severity = parts[1]?.trim() || 'Unknown';
  const type     = parts[2]?.trim() || '';
  const region   = parts[3]?.trim() || '';

  // Time: "12-May 22:41 > 23:20" (clearance has two times) or "12-May 23:34 > OPEN"
  const timeMatch = text.match(/Time:\s*(.+?)\s*>\s*(.+)/i);
  const timeStart = timeMatch?.[1]?.trim() || '';
  const timeEnd   = timeMatch?.[2]?.trim() || '';

  // For clearance: timeEnd is the resolution time (e.g. "23:20"), not a keyword
  const isResolved   = clearance || (timeEnd && !/^OPEN$/i.test(timeEnd));
  const resolvedTime = isResolved && !/^OPEN$/i.test(timeEnd) ? timeEnd : null;

  // Full start timestamp for downtime calc (e.g. "12-May 22:41")
  const downtimeMinutes = clearance && timeStart && resolvedTime
    ? calcDowntimeMinutes(timeStart, `${timeStart.split(' ')[0]} ${resolvedTime}`)
    : null;

  const coordMatch = text.match(/maps\.google\.com\/maps\?q=([\d.]+),([\d.]+)/i);

  const siLine = text.match(/(2G|3G|4G|5G)[^\n]*/);
  const si = {};
  if (siLine) {
    ['2G','3G','4G','5G'].forEach(t => {
      const m = siLine[0].match(new RegExp(t + ':\\s*(\\d+)'));
      if (m) si[t] = m[1];
    });
  }

  return {
    id:               Date.now() + '_' + Math.random().toString(36).slice(2,6),
    messageType:      clearance ? 'CLEARANCE' : 'NOTIFICATION',
    severity,
    type,
    region,
    description:      getSection(text, 'Description'),
    timeStart,
    timeEnd,
    status:           isResolved ? 'CLOSED' : 'OPEN',
    resolvedTime,
    downtimeMinutes,
    downtimeFormatted: formatDowntime(downtimeMinutes),
    serviceImpact:    si,
    bc:               (text.match(/BC:\s*(.+)/i)?.[1] || '').trim(),
    coverage:         getSection(text, 'Coverage Impact'),
    rootCause:        getSection(text, 'Root Cause'),
    action:           getSection(text, 'Action & Recommendation'),
    sr:               text.match(/SR\s*\(Last 30 days\)\s*:\s*(\d+)/i)?.[1] || '',
    coordinates:      coordMatch ? `${coordMatch[1]},${coordMatch[2]}` : '',
    siteIds:          extractSiteIds(text),
    raw,
    receivedAt:       new Date().toISOString(),
    clearedAt:        isResolved ? new Date().toISOString() : null,
  };
}

// When a Clearance arrives, find the matching open Notification and close it
function applyClearance(clearanceRecord, data) {
  const cSites = clearanceRecord.siteIds;
  let matched  = false;

  for (let i = 0; i < data.length; i++) {
    const inc = data[i];
    if (inc.messageType !== 'NOTIFICATION') continue;
    if (inc.status === 'CLOSED') continue;

    // Match if any siteId overlaps
    const overlap = inc.siteIds.some(s => cSites.includes(s));
    if (overlap) {
      data[i] = {
        ...data[i],
        status:            'CLOSED',
        clearedAt:         clearanceRecord.receivedAt,
        clearanceId:       clearanceRecord.id,
        resolvedTime:      clearanceRecord.resolvedTime,
        downtimeMinutes:   clearanceRecord.downtimeMinutes,
        downtimeFormatted: clearanceRecord.downtimeFormatted,
        rootCause:         clearanceRecord.rootCause || data[i].rootCause,
        action:            clearanceRecord.action    || data[i].action,
      };
      matched = true;
      // don't break — close ALL matching open incidents
    }
  }
  return matched;
}

// ── routes ────────────────────────────────────────────────────────────────────

app.get('/', (req, res) => {
  const data = loadData();
  res.json({
    status: 'NOC Webhook Server running ✅',
    total: data.length,
    open:  data.filter(i => i.status === 'OPEN').length,
    closed: data.filter(i => i.status === 'CLOSED').length,
  });
});

app.post('/webhook', (req, res) => {
  const secret = req.query.secret || req.headers['x-secret'];
  if (secret !== SECRET) return res.status(401).json({ error: 'Unauthorized' });

  const body    = req.body;
  const rawText = body.content || body.message || body.text || body.body || '';
  if (!rawText) return res.status(400).json({ error: 'No message content' });

  // Accept Clearance OR Notification messages
  const isNoc = /^(clearance|notification)\s*:/i.test(rawText.trim()) ||
    rawText.toLowerCase().includes('tech-noc') ||
    rawText.toLowerCase().includes('technology noc');

  if (!isNoc) return res.json({ skipped: true, reason: 'Not a NOC message' });

  const record = parseSMS(rawText);
  record.sender  = body.sender || body.from || '';
  record.contact = body.contact || '';

  const data = loadData();

  if (record.messageType === 'CLEARANCE') {
    const matched = applyClearance(record, data);
    data.unshift(record); // store clearance too for audit trail
    saveData(data);
    console.log(`[CLEARANCE] Sites: ${record.siteIds.join(', ')} | Matched open incident: ${matched}`);
    res.json({ ok: true, type: 'CLEARANCE', matched, id: record.id });
  } else {
    data.unshift(record);
    saveData(data);
    console.log(`[NOTIFICATION] ${record.severity} - ${record.region} - Sites: ${record.siteIds.join(', ')}`);
    res.json({ ok: true, type: 'NOTIFICATION', id: record.id, siteIds: record.siteIds });
  }
});

app.get('/incidents', (req, res) => {
  res.json(loadData());
});

app.get('/sites', (req, res) => {
  const data = loadData();
  const map  = {};
  // Only count NOTIFICATION messages as outages (not clearances)
  data.filter(i => i.messageType !== 'CLEARANCE').forEach(inc => {
    (inc.siteIds.length ? inc.siteIds : ['UNKNOWN']).forEach(sid => {
      if (!map[sid]) map[sid] = { siteId: sid, count: 0, openCount: 0, incidents: [] };
      map[sid].count++;
      if (inc.status === 'OPEN') map[sid].openCount++;
      map[sid].incidents.push(inc);
    });
  });
  res.json(Object.values(map).sort((a, b) => b.count - a.count));
});

app.get('/sites/:siteId', (req, res) => {
  const data = loadData();
  const sid  = req.params.siteId.toUpperCase();
  const incidents = data.filter(i =>
    i.messageType !== 'CLEARANCE' &&
    i.siteIds.map(s => s.toUpperCase()).includes(sid)
  );
  if (!incidents.length) return res.status(404).json({ error: 'Site not found' });
  res.json({ siteId: sid, count: incidents.length, lastSeen: incidents[0].receivedAt, incidents });
});

app.delete('/incidents/:id', (req, res) => {
  const secret = req.query.secret || req.headers['x-secret'];
  if (secret !== SECRET) return res.status(401).json({ error: 'Unauthorized' });
  const data    = loadData();
  const updated = data.filter(i => i.id !== req.params.id);
  saveData(updated);
  res.json({ ok: true, deleted: data.length - updated.length });
});

// Test: simulate a Notification then a Clearance
app.post('/test', (req, res) => {
  const type = req.query.type || 'notification';

  const notif = `Notification : Major : CR : Riyadh
====================
Description:
1 Physical site and 2 Business customers connected to FON C0801 (P3) are down

Power Source: Actual: SEC / Design: SEC

Service Impact:
2G: 1, 3G: 1, 4G: 1, 5G: 1
BC: 2 (VIP 1)

Time: 12-May 23:34 > OPEN

Root Cause:
Under investigation

Action & Recommendation:
Escalated to EM and OSP

SR (Last 30 days): 0

Coverage Impact:
Riyadh(.8%)

Coordinates:
http://maps.google.com/maps?q=24.7113,46.7659

Technology NOC
0560311500`;

  const clear = `Clearance: Major: CR: Riyadh
====================
Description:
FON C0801 (P3) is back up.

Power Source: Actual: SEC / Design: SEC

Service Impact:
2G: 1, 3G: 1, 4G: 1, 5G: 1
BC: 2 (VIP 1)

Time: 12-May 23:34 > 00:15

Root Cause:
Power issue resolved by EM team.

Action & Recommendation:
Site restored, monitoring.

SR (Last 30 days): 0

Coordinates:
http://maps.google.com/maps?q=24.7113,46.7659

Technology NOC
0560311500`;

  const raw    = type === 'clearance' ? clear : notif;
  const record = parseSMS(raw);
  const data   = loadData();

  if (record.messageType === 'CLEARANCE') {
    applyClearance(record, data);
    data.unshift(record);
  } else {
    data.unshift(record);
  }
  saveData(data);
  res.json({ ok: true, type: record.messageType, record });
});

app.listen(PORT, () => console.log(`NOC Server on port ${PORT}`));
