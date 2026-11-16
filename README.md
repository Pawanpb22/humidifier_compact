# humidifier_compact
#include <WiFi.h>
#include <WebServer.h>
#include "DHT.h"
#define MIST 19
#define FAN 18
DHT dht(4, DHT22);
WebServer srv(80);
float lo = 80, hi = 85, h = NAN, t = NAN, tgt = 80, startRH = NAN;
int mM = 0, fM = 0;  // 0 auto, 1 on, 2 off
bool mist, fan, tAct, tDone;
unsigned long last, tStart, tEl;
const char* MN[] = {"auto", "on", "off"};

void relay(int p, bool on) { on ? (pinMode(p, OUTPUT), digitalWrite(p, LOW)) : pinMode(p, INPUT); }

void control() {
  bool ok = !isnan(h) && !isnan(t);
  mist = ok && (tAct ? h < tgt : mM == 2 ? false : mM == 1 ? h < hi : h < lo);
  fan = ok && (fM == 2 ? false : fM == 1 ? true : h > hi);
  relay(MIST, mist); relay(FAN, fan);
}

const char PAGE[] PROGMEM = R"x(<!DOCTYPE html><html><head><meta name=viewport content="width=device-width,initial-scale=1"><title>Smart Humidity Control</title><style>
body{font-family:Arial;text-align:center;background:#f3f5f7;padding:15px}.box{max-width:500px;margin:auto;padding:20px;background:#fff;border:1px solid #ddd;border-radius:15px}
.value{font-size:32px;margin:8px}.item{padding:12px;margin:10px;background:#eef2f5;border-radius:8px}
button{padding:11px 17px;margin:4px;border:1px solid #888;border-radius:8px;background:#fff;font-size:15px;cursor:pointer}
button.sel,.b{background:#2d6cdf;color:#fff;border-color:#2d6cdf}input{padding:10px;width:85px;font-size:17px;text-align:center;border:1px solid #aaa;border-radius:7px}
.timer{font-size:34px;font-weight:bold;margin:12px;color:#2d6cdf}.small{font-size:14px;color:#555}</style></head><body><div class=box><h1>Smart Humidity Control</h1>
<div class=item><h3>Temperature</h3><div class=value id=t>--</div></div>
<div class=item><h3>Humidity</h3><div class=value id=h>--</div></div>
<div class=item><h3>Mist Maker: <span id=mState>--</span></h3><span id=mist></span></div>
<div class=item><h3>Exhaust Fan: <span id=fState>--</span></h3><span id=fan></span></div>
<div class=item><h3>Adjust Automatic Thresholds</h3><p>Mist ON below lower, fan ON above upper.</p>
<p><b>Mist ON threshold (%)</b></p><input type=number id=lo min=1 max=98 step=.5>
<p><b>Fan ON threshold (%)</b></p><input type=number id=hi min=2 max=99 step=.5><br><br>
<button class=b onclick="api('/settings?lower='+lo.value+'&upper='+hi.value,'thMsg')">APPLY THRESHOLDS</button><p id=thMsg class=small></p></div><hr>
<h2>Humidity Rise Test</h2><div class=item><p><b>Starting Humidity</b></p><div class=value id=sRH>--</div>
<p><b>Target Humidity (%)</b></p><input type=number id=tg value=80 min=1 max=99 step=.5>
<p><b>Elapsed Test Time</b></p><div class=timer id=tm>00:00:00</div><p class=small>Test Status: <span id=st>NOT STARTED</span></p>
<button class=b style="background:#198754" onclick="api('/test?action=start&target='+tg.value,'msg')">START TEST</button>
<button class=b style="background:#dc3545" onclick="api('/test?action=stop','msg')">STOP TEST</button><p id=msg></p></div>
<p>Sensor sampling interval: 2.5 seconds</p><p id=err style="color:red"></p></div><script>
const $=i=>document.getElementById(i),M=['auto','on','off'];
['mist','fan'].forEach(d=>$(d).innerHTML=M.map(m=>`<button id=${d}-${m} onclick="api('/cmd?dev=${d}&mode=${m}')">${m.toUpperCase()}</button>`).join(''));
const f=ms=>{let s=ms/1000|0;return[s/3600|0,s%3600/60|0,s%60].map(v=>String(v).padStart(2,'0')).join(':')};
function R(d){
$('t').textContent=d.ok?d.temp.toFixed(1)+' °C':'--';$('h').textContent=d.ok?d.hum.toFixed(1)+' %':'--';
$('err').textContent=d.ok?'':'Sensor error';$('mState').textContent=d.mist?'ON':'OFF';$('fState').textContent=d.fan?'ON':'OFF';
if(document.activeElement!=$('lo'))$('lo').value=d.lower.toFixed(1);if(document.activeElement!=$('hi'))$('hi').value=d.upper.toFixed(1);
$('thMsg').textContent='Current: Mist ON below '+d.lower.toFixed(1)+'% | Fan ON above '+d.upper.toFixed(1)+'%';
M.forEach(m=>{$('mist-'+m).className=d.mistMode==m?'sel':'';$('fan-'+m).className=d.fanMode==m?'sel':''});
$('sRH').textContent=d.testStarted?d.startRH.toFixed(1)+' %':'--';$('tm').textContent=f(d.testElapsed);
$('st').textContent=d.testActive?'RUNNING':d.testComplete?'TARGET REACHED':d.testStarted?'STOPPED MANUALLY':'NOT STARTED';
$('msg').textContent=d.testActive?'Target: '+d.target.toFixed(1)+'% RH':d.testComplete?'Humidity target reached successfully!':'';}
function api(u,e){fetch(u).then(r=>r.ok?r.json():r.text().then(x=>{throw Error(x)})).then(R).catch(x=>{if(e)$(e).textContent=x.message})}
setInterval(()=>api('/status'),1000);api('/status');
</script></body></html>)x";

String json() {
  bool ok = !isnan(h) && !isnan(t);
  char b[320];
  snprintf(b, sizeof b,
    "{\"ok\":%d,\"temp\":%.1f,\"hum\":%.1f,\"mist\":%d,\"fan\":%d,\"lower\":%.1f,\"upper\":%.1f,"
    "\"mistMode\":\"%s\",\"fanMode\":\"%s\",\"testActive\":%d,\"testComplete\":%d,\"testStarted\":%d,"
    "\"startRH\":%.1f,\"target\":%.1f,\"testElapsed\":%lu}",
    ok, ok ? t : 0, ok ? h : 0, mist, fan, lo, hi, MN[mM], MN[fM], tAct, tDone, !isnan(startRH),
    isnan(startRH) ? 0 : startRH, tgt, tAct ? millis() - tStart : tEl);
  return b;
}

void done() { control(); srv.send(200, "application/json", json()); }
void bad(const char* m) { srv.send(400, "text/plain", m); }

void setup() {
  Serial.begin(115200);
  relay(MIST, 0); relay(FAN, 0);
  dht.begin();
  WiFi.softAP("ESP32-Humidity", "12345678");
  Serial.println(WiFi.softAPIP());
  srv.on("/", [] { srv.send_P(200, "text/html", PAGE); });
  srv.on("/status", [] { srv.send(200, "application/json", json()); });
  srv.on("/cmd", [] {
    String d = srv.arg("dev"), m = srv.arg("mode");
    int v = m == "auto" ? 0 : m == "on" ? 1 : m == "off" ? 2 : -1;
    if (v < 0) return bad("Bad mode");
    if (d == "mist") mM = v; else if (d == "fan") fM = v; else return bad("Bad device");
    done();
  });
  srv.on("/settings", [] {
    if (!srv.hasArg("lower") || !srv.hasArg("upper")) return bad("Enter both thresholds");
    float l = srv.arg("lower").toFloat(), u = srv.arg("upper").toFloat();
    if (l <= 0 || l >= 99 || u <= 0 || u >= 100) return bad("Thresholds must be between 1 and 99");
    if (l >= u) return bad("Mist threshold must be lower than fan threshold");
    lo = l; hi = u;
    done();
  });
  srv.on("/test", [] {
    String a = srv.arg("action");
    if (a == "start") {
      float g = srv.arg("target").toFloat();
      if (isnan(h) || isnan(t)) return bad("Sensor not ready");
      if (g <= 0 || g >= 100) return bad("Invalid target");
      if (g <= h) return bad("Target must be higher than starting humidity");
      startRH = h; tgt = g; tStart = millis(); tEl = 0;
      tAct = true; tDone = false; mM = fM = 0;
    } else if (a == "stop") {
      if (tAct) tEl = millis() - tStart;
      tAct = tDone = false; mM = 2;
    } else return bad("Invalid action");
    done();
  });
  srv.begin();
}

void loop() {
  srv.handleClient();
  if (millis() - last < 2500) return;
  last = millis();
  h = dht.readHumidity(); t = dht.readTemperature();
  control();
  if (tAct && !isnan(h)) {
    tEl = millis() - tStart;
    if (h >= tgt) { tAct = false; tDone = true; control(); Serial.println("TEST TARGET REACHED!"); }
  }
  Serial.printf("Hum: %.1f%% | Temp: %.1fC | Mist: %d | Fan: %d | Test: %d\n", h, t, mist, fan, tAct);
}
