<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover,user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="theme-color" content="#0b0f1a">
<title>BodyLog</title>
<link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Mono:wght@400;700&family=DM+Sans:wght@400;500;600&display=swap" rel="stylesheet">
<style>
*{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent}
html,body,#root{height:100%;overflow:hidden;background:#0b0f1a}
body{font-family:'DM Sans',sans-serif;color:#fff}
input,select,textarea{outline:none;border:1px solid rgba(255,255,255,.1);background:rgba(255,255,255,.07);color:#fff;font-family:'DM Sans',sans-serif;border-radius:10px;padding:10px 14px;font-size:15px;width:100%;-webkit-appearance:none}
input[type=number]{-moz-appearance:textfield}
input[type=number]::-webkit-outer-spin-button,input[type=number]::-webkit-inner-spin-button{-webkit-appearance:none}
input::placeholder,textarea::placeholder{color:rgba(255,255,255,.3)}
input[type=date]{color-scheme:dark}
select option{background:#1e2536}
textarea{resize:none;line-height:1.6}
button{font-family:'DM Sans',sans-serif;cursor:pointer;-webkit-appearance:none}
::-webkit-scrollbar{width:3px}
::-webkit-scrollbar-thumb{background:rgba(255,255,255,.12);border-radius:4px}
@keyframes spin{to{transform:rotate(360deg)}}
@keyframes fadeUp{from{opacity:0;transform:translateY(12px)}to{opacity:1;transform:translateY(0)}}
.fade-up{animation:fadeUp .3s ease both}
</style>
</head>
<body>
<div id="root"></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
<script type="text/babel" data-presets="react">
const {useState,useRef,useEffect}=React;

/* ── storage ── */
const LS={
  get:(k,d)=>{try{const v=localStorage.getItem(k);return v?JSON.parse(v):d}catch{return d}},
  set:(k,v)=>{try{localStorage.setItem(k,JSON.stringify(v))}catch{}}
};

/* ── helpers ── */
const todayStr=()=>new Date().toISOString().split('T')[0];

function calcBMR(p){
  const w=parseFloat(p.weight)||70,h=parseFloat(p.height)||170,
        a=parseFloat(p.age)||25,bf=parseFloat(p.bodyFat);
  if(bf)return Math.round(370+21.6*w*(1-bf/100));
  return p.gender==='male'
    ?Math.round(88.362+13.397*w+4.799*h-5.677*a)
    :Math.round(447.593+9.247*w+3.098*h-4.33*a);
}
function calcTDEE(bmr,act){
  const m={sedentary:1.2,light:1.375,moderate:1.55,active:1.725,veryActive:1.9};
  return Math.round(bmr*(m[act]||1.2));
}

const SCORE={
  excellent:{label:'優質',color:'#4ade80',bg:'rgba(74,222,128,.15)'},
  good:     {label:'良好',color:'#facc15',bg:'rgba(250,204,21,.15)'},
  poor:     {label:'少吃',color:'#f87171',bg:'rgba(248,113,113,.15)'}
};

const MEALS=[
  {id:'breakfast',label:'早餐',icon:'🌅',color:'#facc15'},
  {id:'lunch',    label:'午餐',icon:'🌤',color:'#fb923c'},
  {id:'dinner',   label:'晚餐',icon:'🌙',color:'#818cf8'},
  {id:'snack',    label:'點心',icon:'🍎',color:'#4ade80'}
];

const ACT={sedentary:'久坐',light:'輕度',moderate:'中度',active:'積極',veryActive:'非常積極'};

/* ── image compress ── */
function compressImage(file,maxPx,quality){
  return new Promise(function(res,rej){
    var url=URL.createObjectURL(file);
    var img=new Image();
    img.onload=function(){
      var w=img.width,h=img.height;
      if(w>maxPx||h>maxPx){
        if(w>h){h=Math.round(h*maxPx/w);w=maxPx;}
        else{w=Math.round(w*maxPx/h);h=maxPx;}
      }
      var c=document.createElement('canvas');
      c.width=w;c.height=h;
      c.getContext('2d').drawImage(img,0,0,w,h);
      URL.revokeObjectURL(url);
      var d=c.toDataURL('image/jpeg',quality||0.82);
      res({base64:d.split(',')[1],mimeType:'image/jpeg'});
    };
    img.onerror=function(){rej(new Error('圖片載入失敗'));};
    img.src=url;
  });
}

/* ── JSON extract ── */
function extractJSON(text){
  var clean=text.replace(/```json\s*/gi,'').replace(/```\s*/g,'').trim();
  try{return JSON.parse(clean)}catch(e){}
  var s=clean.indexOf('{'),e2=clean.lastIndexOf('}');
  if(s!==-1&&e2>s){try{return JSON.parse(clean.slice(s,e2+1))}catch(e){}}
  throw new Error('無法解析 AI 回應');
}

/* ── API calls ── */
async function callClaude(apiKey,messages,maxTokens){
  var res=await fetch('https://api.anthropic.com/v1/messages',{
    method:'POST',
    headers:{
      'Content-Type':'application/json',
      'x-api-key':apiKey,
      'anthropic-version':'2023-06-01',
      'anthropic-dangerous-direct-browser-access':'true'
    },
    body:JSON.stringify({model:'claude-sonnet-4-20250514',max_tokens:maxTokens||1000,messages:messages})
  });
  var data=await res.json();
  if(data.error)throw new Error(data.error.message);
  return (data.content||[]).map(function(i){return i.text||'';}).join('');
}

async function analyzeFood(file,apiKey){
  var compressed=await compressImage(file,1600,0.82);
  var prompt='You are a nutritionist. Analyze this meal photo.\nReturn ONLY valid JSON, no markdown:\n{"items":[{"name":"雞胸肉","cal":165,"p":31,"c":0,"f":3.6,"score":"excellent"}],"total":{"cal":165,"p":31,"c":0,"f":3.6},"advice":"一句建議（繁體中文）"}\nscore must be excellent/good/poor. Use Traditional Chinese food names.';
  var text=await callClaude(apiKey,[{role:'user',content:[
    {type:'image',source:{type:'base64',media_type:compressed.mimeType,data:compressed.base64}},
    {type:'text',text:prompt}
  ]}],1500);
  return extractJSON(text);
}

async function analyzeFoodText(desc,apiKey){
  var prompt='你是專業營養師。用戶描述吃了：「'+desc+'」\n請估算每樣食物的營養。只輸出 JSON，不加 markdown：\n{"items":[{"name":"食物名","cal":165,"p":31,"c":0,"f":3.6,"score":"excellent"}],"total":{"cal":165,"p":31,"c":0,"f":3.6},"advice":"一句建議（繁體中文）"}\nscore 只能是 excellent/good/poor。';
  var text=await callClaude(apiKey,[{role:'user',content:prompt}],1500);
  return extractJSON(text);
}

async function analyzeInBody(file,apiKey){
  var compressed=await compressImage(file,1920,0.85);
  var prompt='You are a body composition expert. Read this InBody report.\nReturn ONLY valid JSON, null for missing values:\n{"weight":70.5,"bodyFat":18.2,"muscleMass":30.1,"fatMass":12.8,"bmi":22.1,"visceralFat":5,"bmr":1650,"bodyWater":38.2,"boneMass":2.8,"assessment":"評估（繁體中文）","suggestions":["建議1","建議2","建議3"]}\nAll numbers must be numeric. Use Traditional Chinese.';
  var text=await callClaude(apiKey,[{role:'user',content:[
    {type:'image',source:{type:'base64',media_type:compressed.mimeType,data:compressed.base64}},
    {type:'text',text:prompt}
  ]}],1000);
  return extractJSON(text);
}

/* ── small components ── */
function Spinner(props){
  var color=props.color||'#818cf8';
  return React.createElement('div',{style:{display:'flex',flexDirection:'column',alignItems:'center',gap:12,padding:'20px 0'}},
    React.createElement('div',{style:{width:34,height:34,borderRadius:'50%',border:'3px solid rgba(255,255,255,.08)',borderTopColor:color,animation:'spin .8s linear infinite'}}),
    React.createElement('div',{style:{fontSize:12,color:'rgba(255,255,255,.4)'}},props.label||'AI 分析中...')
  );
}

function Ring(props){
  var value=props.value||0,max=props.max||1,color=props.color,size=props.size||64,stroke=props.stroke||7;
  var r=(size-stroke)/2,circ=2*Math.PI*r,pct=Math.min(value/max,1);
  return (
    <div style={{display:'flex',flexDirection:'column',alignItems:'center',gap:4}}>
      <svg width={size} height={size}>
        <circle cx={size/2} cy={size/2} r={r} fill='none' stroke='rgba(255,255,255,.08)' strokeWidth={stroke}/>
        <circle cx={size/2} cy={size/2} r={r} fill='none' stroke={color} strokeWidth={stroke}
          strokeDasharray={circ} strokeDashoffset={circ*(1-pct)} strokeLinecap='round'
          transform={'rotate(-90 '+(size/2)+' '+(size/2)+')'}
          style={{transition:'stroke-dashoffset .5s ease'}}/>
        <text x={size/2} y={size/2+5} textAnchor='middle' fill='white' fontSize={10} fontFamily='DM Mono,monospace' fontWeight='700'>
          {Math.round(pct*100)}%
        </text>
      </svg>
      <span style={{fontSize:11,color:'rgba(255,255,255,.6)',textAlign:'center'}}>{props.label}</span>
      {props.sub&&<span style={{fontSize:10,color,fontFamily:'DM Mono,monospace'}}>{props.sub}</span>}
    </div>
  );
}

function MiniChart(props){
  var data=props.data,key=props.valueKey,color=props.color;
  if(!data||data.length<2)return null;
  var vals=data.map(function(d){return d[key];}).filter(function(v){return v!=null;});
  if(vals.length<2)return null;
  var minV=Math.min.apply(null,vals),maxV=Math.max.apply(null,vals),range=maxV-minV||1;
  var W=280,H=56;
  var filtered=data.filter(function(d){return d[key]!=null;});
  var pts=filtered.map(function(d,i){
    var x=10+(i/Math.max(filtered.length-1,1))*(W-20);
    var y=H-6-((d[key]-minV)/range)*(H-14);
    return x+','+y;
  }).join(' ');
  return (
    <svg viewBox={'0 0 '+W+' '+H} style={{width:'100%',height:56}}>
      <polyline fill='none' stroke={color} strokeWidth={2} points={pts} strokeLinejoin='round' opacity={0.8}/>
      {filtered.map(function(d,i){
        var x=10+(i/Math.max(filtered.length-1,1))*(W-20);
        var y=H-6-((d[key]-minV)/range)*(H-14);
        return <circle key={i} cx={x} cy={y} r={3} fill={color}/>;
      })}
    </svg>
  );
}

function Card(props){
  return (
    <div style={Object.assign({background:'rgba(255,255,255,.03)',border:'1px solid rgba(255,255,255,.07)',borderRadius:20,padding:20},props.style||{})}>
      {props.children}
    </div>
  );
}

function SectionTitle(props){
  return <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:props.mb!=null?props.mb:14,letterSpacing:'0.05em',textTransform:'uppercase',fontWeight:600}}>{props.children}</div>;
}

function DiffBadge(props){
  var val=props.val,unit=props.unit||'',good=props.good||'down';
  if(val==null||val===0)return <span style={{fontSize:11,color:'rgba(255,255,255,.25)'}}>—</span>;
  var up=val>0;
  var positive=(good==='down')?!up:up;
  return <span style={{fontSize:11,color:positive?'#4ade80':'#f87171',fontFamily:'DM Mono,monospace'}}>{up?'+':''}{val}{unit}</span>;
}

function UploadZone(props){
  var ref=useRef();
  return (
    <div onClick={function(){ref.current.click();}}
      style={{border:'2px dashed rgba(255,255,255,.14)',borderRadius:14,padding:'28px 16px',textAlign:'center',cursor:'pointer',background:'rgba(255,255,255,.02)'}}>
      <input ref={ref} type='file' accept='image/*' style={{display:'none'}}
        onChange={function(e){var f=e.target.files[0];if(f)props.onFile(f);e.target.value='';}}/>
      <div style={{fontSize:34,marginBottom:8}}>{props.icon}</div>
      <div style={{fontSize:13,color:'rgba(255,255,255,.7)',fontWeight:500}}>{props.label}</div>
      <div style={{fontSize:11,color:'rgba(255,255,255,.3)',marginTop:5}}>點擊拍照或從相簿選取</div>
    </div>
  );
}

/* ── InBody History ── */
function InBodyHistory(props){
  var logs=props.logs,onDelete=props.onDelete;
  var [expanded,setExpanded]=useState(null);
  var [confirmDel,setConfirmDel]=useState(null);
  if(!logs.length)return(
    <div style={{color:'rgba(255,255,255,.3)',fontSize:13,textAlign:'center',padding:'32px 0',lineHeight:2}}>
      尚無量測記錄<br/><span style={{fontSize:12}}>上傳 InBody 報告後點「儲存記錄」</span>
    </div>
  );
  var chron=[...logs].sort(function(a,b){return a.date.localeCompare(b.date)||a.id-b.id;});
  var desc=[...logs].sort(function(a,b){return b.date.localeCompare(a.date)||b.id-a.id;});
  var chartFields=[
    {key:'weight',label:'體重',unit:'kg',color:'#a78bfa',good:'down'},
    {key:'bodyFat',label:'體脂率',unit:'%',color:'#f87171',good:'down'},
    {key:'muscleMass',label:'骨骼肌',unit:'kg',color:'#4ade80',good:'up'},
    {key:'bmi',label:'BMI',unit:'',color:'#facc15',good:'down'}
  ];
  var diffFields=[
    {key:'weight',label:'體重',unit:'kg',good:'down'},
    {key:'bodyFat',label:'體脂率',unit:'%',good:'down'},
    {key:'muscleMass',label:'骨骼肌',unit:'kg',good:'up'},
    {key:'fatMass',label:'體脂肪',unit:'kg',good:'down'},
    {key:'bmi',label:'BMI',unit:'',good:'down'},
    {key:'visceralFat',label:'內臟脂肪',unit:'級',good:'down'}
  ];
  var cur=desc[0],prv=desc[1];
  return (
    <div style={{display:'flex',flexDirection:'column',gap:14}}>
      <Card>
        <SectionTitle>趨勢圖（共 {logs.length} 筆）</SectionTitle>
        <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:16}}>
          {chartFields.map(function(f){
            var pts=chron.filter(function(d){return d[f.key]!=null;});
            if(pts.length<2)return(
              <div key={f.key}>
                <div style={{fontSize:11,color:'rgba(255,255,255,.35)'}}>{f.label}</div>
                <div style={{fontSize:10,color:'rgba(255,255,255,.2)',marginTop:4}}>需至少 2 筆</div>
              </div>
            );
            var last=pts[pts.length-1][f.key],prev=pts[pts.length-2][f.key];
            var diff=+(last-prev).toFixed(1);
            return(
              <div key={f.key}>
                <div style={{display:'flex',justifyContent:'space-between',alignItems:'center',marginBottom:2}}>
                  <span style={{fontSize:11,color:'rgba(255,255,255,.45)'}}>{f.label}</span>
                  <DiffBadge val={diff} unit={f.unit} good={f.good}/>
                </div>
                <div style={{fontFamily:'DM Mono,monospace',fontSize:17,color:f.color,fontWeight:700,marginBottom:4}}>
                  {last}<span style={{fontSize:10}}>{f.unit}</span>
                </div>
                <MiniChart data={pts} valueKey={f.key} color={f.color}/>
              </div>
            );
          })}
        </div>
      </Card>
      {desc.length>=2&&(
        <Card>
          <SectionTitle>最新 vs 上次差異</SectionTitle>
          <div style={{fontSize:11,color:'rgba(255,255,255,.3)',marginBottom:10}}>{prv.date} → {cur.date}</div>
          <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:8}}>
            {diffFields.filter(function(f){return cur[f.key]!=null&&prv[f.key]!=null;}).map(function(f){
              var diff=+(cur[f.key]-prv[f.key]).toFixed(1);
              return(
                <div key={f.key} style={{background:'rgba(255,255,255,.04)',borderRadius:10,padding:'10px 12px'}}>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)'}}>{f.label}</div>
                  <div style={{fontFamily:'DM Mono,monospace',fontSize:14,color:'white',fontWeight:700,marginTop:2}}>
                    {cur[f.key]}<span style={{fontSize:10,color:'rgba(255,255,255,.35)'}}>{f.unit}</span>
                  </div>
                  <DiffBadge val={diff} unit={f.unit} good={f.good}/>
                </div>
              );
            })}
          </div>
        </Card>
      )}
      <Card>
        <SectionTitle>所有記錄</SectionTitle>
        {desc.map(function(log){
          var isOpen=expanded===log.id,isDel=confirmDel===log.id;
          var t=new Date(log.id);
          var timeStr=isNaN(t.getTime())?'':String(t.getHours()).padStart(2,'0')+':'+String(t.getMinutes()).padStart(2,'0');
          return(
            <div key={log.id} style={{borderBottom:'1px solid rgba(255,255,255,.05)'}}>
              <div style={{display:'flex',alignItems:'center',padding:'10px 0',gap:8}}>
                <div onClick={function(){setExpanded(isOpen?null:log.id);}} style={{flex:1,cursor:'pointer'}}>
                  <div style={{display:'flex',alignItems:'center',gap:8}}>
                    <span style={{fontSize:13,fontWeight:600}}>{log.date}</span>
                    {timeStr&&<span style={{fontSize:10,color:'rgba(255,255,255,.3)',fontFamily:'DM Mono,monospace'}}>{timeStr}</span>}
                  </div>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginTop:2}}>
                    {log.weight?log.weight+'kg':''}{log.bodyFat?' · 體脂'+log.bodyFat+'%':''}{log.muscleMass?' · 肌肉'+log.muscleMass+'kg':''}
                  </div>
                </div>
                {!isDel
                  ?<button onClick={function(){setConfirmDel(log.id);}} style={{background:'none',border:'none',color:'rgba(255,255,255,.2)',fontSize:18,padding:'4px 6px'}}>×</button>
                  :<div style={{display:'flex',gap:6,alignItems:'center'}}>
                    <span style={{fontSize:11,color:'#f87171'}}>確定?</span>
                    <button onClick={function(){onDelete(log.id);setConfirmDel(null);setExpanded(null);}} style={{background:'rgba(248,113,113,.2)',border:'1px solid rgba(248,113,113,.4)',borderRadius:8,color:'#f87171',fontSize:11,padding:'3px 8px'}}>刪除</button>
                    <button onClick={function(){setConfirmDel(null);}} style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:8,color:'rgba(255,255,255,.5)',fontSize:11,padding:'3px 8px'}}>取消</button>
                  </div>
                }
                <span onClick={function(){setExpanded(isOpen?null:log.id);}} style={{fontSize:11,color:'rgba(255,255,255,.25)',cursor:'pointer'}}>{isOpen?'▲':'▼'}</span>
              </div>
              {isOpen&&(
                <div style={{paddingBottom:12}}>
                  {log.assessment&&<div style={{background:'rgba(129,140,248,.08)',border:'1px solid rgba(129,140,248,.15)',borderRadius:10,padding:'8px 12px',fontSize:12,color:'#a78bfa',marginBottom:10,lineHeight:1.5}}>🔍 {log.assessment}</div>}
                  <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:6,marginBottom:8}}>
                    {[
                      {label:'體重',val:log.weight,unit:'kg'},
                      {label:'體脂率',val:log.bodyFat,unit:'%'},
                      {label:'骨骼肌',val:log.muscleMass,unit:'kg'},
                      {label:'體脂肪',val:log.fatMass,unit:'kg'},
                      {label:'BMI',val:log.bmi,unit:''},
                      {label:'基礎代謝',val:log.bmr,unit:'kcal'},
                      {label:'內臟脂肪',val:log.visceralFat,unit:'級'},
                      {label:'體水分',val:log.bodyWater,unit:'kg'}
                    ].filter(function(x){return x.val!=null;}).map(function(x,j){
                      return(
                        <div key={j} style={{background:'rgba(255,255,255,.03)',borderRadius:8,padding:'8px 10px'}}>
                          <div style={{fontSize:10,color:'rgba(255,255,255,.3)'}}>{x.label}</div>
                          <div style={{fontFamily:'DM Mono,monospace',fontSize:14,color:'#818cf8',fontWeight:700}}>{x.val}<span style={{fontSize:10}}>{x.unit}</span></div>
                        </div>
                      );
                    })}
                  </div>
                  {log.suggestions&&log.suggestions.map(function(s,j){
                    return <div key={j} style={{display:'flex',gap:8,fontSize:12,color:'rgba(255,255,255,.5)',padding:'3px 0'}}><span style={{color:'#818cf8'}}>→</span><span>{s}</span></div>;
                  })}
                </div>
              )}
            </div>
          );
        })}
      </Card>
    </div>
  );
}

/* ── Setup Screen ── */
function SetupScreen(props){
  var [key,setKey]=useState('');
  var [err,setErr]=useState('');
  var [loading,setLoading]=useState(false);
  async function go(){
    var k=key.trim();
    if(!k.startsWith('sk-ant-')){setErr('請輸入正確的 API Key（以 sk-ant- 開頭）');return;}
    setLoading(true);setErr('');
    try{
      await callClaude(k,[{role:'user',content:'hi'}],10);
      LS.set('bl_apikey',k);
      props.onDone(k);
    }catch(e){setErr('驗證失敗：'+e.message);}
    setLoading(false);
  }
  return (
    <div style={{minHeight:'100vh',background:'linear-gradient(160deg,#0b0f1a,#111827)',display:'flex',flexDirection:'column',alignItems:'center',justifyContent:'center',padding:'40px 28px'}}>
      <div style={{fontFamily:'DM Serif Display,serif',fontSize:44,marginBottom:6}}>BodyLog</div>
      <div style={{fontSize:13,color:'rgba(255,255,255,.4)',marginBottom:40}}>AI 驅動的個人健康追蹤</div>
      <div style={{width:'100%',maxWidth:380}}>
        <Card>
          <SectionTitle>Anthropic API Key</SectionTitle>
          <div style={{fontSize:13,color:'rgba(255,255,255,.5)',marginBottom:14,lineHeight:1.7}}>
            前往 <span style={{color:'#818cf8'}}>console.anthropic.com</span> 取得 Key
          </div>
          <input type='password' value={key} onChange={function(e){setKey(e.target.value);}} onKeyDown={function(e){if(e.key==='Enter')go();}} placeholder='sk-ant-api03-...' style={{marginBottom:10}}/>
          {err&&<div style={{fontSize:12,color:'#f87171',marginBottom:10}}>{err}</div>}
          <button onClick={go} disabled={loading}
            style={{width:'100%',background:'linear-gradient(135deg,#818cf8,#6366f1)',border:'none',borderRadius:12,color:'white',padding:14,fontSize:14,fontWeight:600,opacity:loading?0.6:1}}>
            {loading?'驗證中...':'開始使用'}
          </button>
          <div style={{fontSize:11,color:'rgba(255,255,255,.2)',marginTop:10,textAlign:'center'}}>API Key 僅儲存於本機瀏覽器</div>
        </Card>
      </div>
    </div>
  );
}

/* ── Main App ── */
function App(){
  var [apiKey,setApiKey]=useState(function(){return LS.get('bl_apikey','');});
  var [tab,setTab]=useState('dashboard');
  var [profile,setProfile]=useState(function(){return LS.get('bl_profile',{name:'',weight:70,height:170,age:25,gender:'male',bodyFat:'',activity:'moderate',goal:'maintain',waterGoal:2000});});
  var [profileSaved,setProfileSaved]=useState(false);
  var [weightLog,setWeightLog]=useState(function(){return LS.get('bl_weights',[]);});
  var [todayWeight,setTodayWeight]=useState('');
  var [water,setWater]=useState(function(){var d=LS.get('bl_water',{d:'',ml:0});return d.d===todayStr()?d.ml:0;});
  var [diary,setDiary]=useState(function(){return LS.get('bl_diary',{});});
  var [dietDate,setDietDate]=useState(todayStr);
  var [activeMeal,setActiveMeal]=useState(null);
  var [mealMode,setMealMode]=useState('text');
  var [mealText,setMealText]=useState('');
  var [foodImg,setFoodImg]=useState(null);
  var [foodLoading,setFoodLoading]=useState(false);
  var [foodResult,setFoodResult]=useState(null);
  var [foodError,setFoodError]=useState('');
  var [inbodyLogs,setInbodyLogs]=useState(function(){return LS.get('bl_inbody_logs',[]);});
  var [inbodyTab,setInbodyTab]=useState('scan');
  var [ibImg,setIbImg]=useState(null);
  var [ibLoading,setIbLoading]=useState(false);
  var [ibResult,setIbResult]=useState(null);
  var [ibError,setIbError]=useState('');
  var [ibDate,setIbDate]=useState(todayStr);

  useEffect(function(){LS.set('bl_profile',profile);},[profile]);
  useEffect(function(){LS.set('bl_weights',weightLog);},[weightLog]);
  useEffect(function(){LS.set('bl_water',{d:todayStr(),ml:water});},[water]);
  useEffect(function(){LS.set('bl_diary',diary);},[diary]);
  useEffect(function(){LS.set('bl_inbody_logs',inbodyLogs);},[inbodyLogs]);

  var bmr=calcBMR(profile);
  var tdee=calcTDEE(bmr,profile.activity);
  var targetCal=profile.goal==='lose'?tdee-500:profile.goal==='gain'?tdee+300:tdee;
  var sortedW=[...weightLog].sort(function(a,b){return a.d.localeCompare(b.d);});
  var todayW=weightLog.find(function(x){return x.d===todayStr();});

  // diary helpers
  var dayMeals=diary[dietDate]||{breakfast:[],lunch:[],dinner:[],snack:[]};
  var allItems=Object.values(dayMeals).flat();
  var dayCal=allItems.reduce(function(s,f){return s+(f.cal||0);},0);
  var dayP=allItems.reduce(function(s,f){return s+(f.p||0);},0);
  var dayC=allItems.reduce(function(s,f){return s+(f.c||0);},0);
  var dayF=allItems.reduce(function(s,f){return s+(f.f||0);},0);

  var todayDiary=diary[todayStr()]||{breakfast:[],lunch:[],dinner:[],snack:[]};
  var todayItems=Object.values(todayDiary).flat();
  var todayCal=todayItems.reduce(function(s,f){return s+(f.cal||0);},0);
  var todayP=todayItems.reduce(function(s,f){return s+(f.p||0);},0);
  var todayC=todayItems.reduce(function(s,f){return s+(f.c||0);},0);
  var todayF=todayItems.reduce(function(s,f){return s+(f.f||0);},0);

  function closePanel(){setActiveMeal(null);setFoodImg(null);setFoodResult(null);setFoodError('');setMealText('');setMealMode('text');}

  function addItemsToMeal(meal,items){
    setDiary(function(d){
      var prev=d[dietDate]||{breakfast:[],lunch:[],dinner:[],snack:[]};
      var existing=prev[meal]||[];
      var added=items.map(function(x){return Object.assign({},x,{id:Date.now()+Math.random()});});
      var updated=Object.assign({},prev);
      updated[meal]=existing.concat(added);
      return Object.assign({},d,{[dietDate]:updated});
    });
  }
  function removeItem(meal,id){
    setDiary(function(d){
      var prev=d[dietDate]||{breakfast:[],lunch:[],dinner:[],snack:[]};
      var updated=Object.assign({},prev);
      updated[meal]=(prev[meal]||[]).filter(function(x){return x.id!==id;});
      return Object.assign({},d,{[dietDate]:updated});
    });
  }

  async function handleFoodFile(file){
    setFoodImg(URL.createObjectURL(file));
    setFoodResult(null);setFoodError('');setFoodLoading(true);
    try{setFoodResult(await analyzeFood(file,apiKey));}
    catch(e){setFoodError('分析失敗：'+e.message);}
    setFoodLoading(false);
  }
  async function handleFoodText(){
    if(!mealText.trim())return;
    setFoodResult(null);setFoodError('');setFoodLoading(true);
    try{setFoodResult(await analyzeFoodText(mealText,apiKey));}
    catch(e){setFoodError('分析失敗：'+e.message);}
    setFoodLoading(false);
  }
  function confirmAdd(){
    if(!foodResult||!activeMeal)return;
    addItemsToMeal(activeMeal,foodResult.items||[]);
    closePanel();
  }

  async function handleIbFile(file){
    setIbImg(URL.createObjectURL(file));
    setIbResult(null);setIbError('');setIbLoading(true);
    try{
      var r=await analyzeInBody(file,apiKey);
      setIbResult(r);
      if(r.weight)setProfile(function(p){return Object.assign({},p,{weight:r.weight});});
      if(r.bodyFat)setProfile(function(p){return Object.assign({},p,{bodyFat:r.bodyFat});});
    }catch(e){setIbError('讀取失敗：'+e.message);}
    setIbLoading(false);
  }
  function saveIbLog(){
    if(!ibResult)return;
    var entry=Object.assign({},ibResult,{id:Date.now(),date:ibDate});
    setInbodyLogs(function(l){return l.concat([entry]);});
    setIbImg(null);setIbResult(null);setIbDate(todayStr());
    setInbodyTab('history');
  }

  if(!apiKey)return <SetupScreen onDone={function(k){setApiKey(k);}}/>;

  var TABS=[
    {id:'dashboard',icon:'◉',label:'總覽'},
    {id:'diet',icon:'⊕',label:'飲食'},
    {id:'weight',icon:'↕',label:'體重'},
    {id:'water',icon:'◌',label:'水分'},
    {id:'profile',icon:'◈',label:'資料'}
  ];

  return (
    <div style={{height:'100vh',display:'flex',flexDirection:'column',background:'linear-gradient(160deg,#0b0f1a,#111827)',maxWidth:480,margin:'0 auto',position:'relative'}}>

      {/* header */}
      <div style={{padding:'16px 22px 12px',display:'flex',justifyContent:'space-between',alignItems:'flex-end',flexShrink:0}}>
        <div>
          <div style={{fontFamily:'DM Serif Display,serif',fontSize:26}}>{profile.name?'Hi, '+profile.name+' 👋':'BodyLog'}</div>
          <div style={{fontSize:12,color:'rgba(255,255,255,.4)',marginTop:1}}>{new Date().toLocaleDateString('zh-TW',{month:'long',day:'numeric',weekday:'short'})}</div>
        </div>
        <div style={{background:'rgba(129,140,248,.15)',border:'1px solid rgba(129,140,248,.25)',borderRadius:10,padding:'5px 12px',fontSize:11,color:'#818cf8',fontFamily:'DM Mono,monospace'}}>BMR {bmr}</div>
      </div>

      {/* scrollable content */}
      <div style={{flex:1,overflowY:'auto',padding:'0 22px 90px'}}>

        {/* ── DASHBOARD ── */}
        {tab==='dashboard'&&(
          <div style={{display:'flex',flexDirection:'column',gap:14}} className='fade-up'>
            {/* calorie */}
            <div style={{background:'linear-gradient(135deg,rgba(129,140,248,.2),rgba(99,102,241,.07))',border:'1px solid rgba(129,140,248,.22)',borderRadius:20,padding:20}}>
              <div style={{fontSize:12,color:'rgba(255,255,255,.4)',marginBottom:10}}>今日熱量</div>
              <div style={{display:'flex',alignItems:'flex-end',gap:8,marginBottom:12}}>
                <span style={{fontFamily:'DM Mono,monospace',fontSize:40,fontWeight:700,color:'#818cf8',lineHeight:1}}>{todayCal}</span>
                <span style={{fontSize:13,color:'rgba(255,255,255,.3)',paddingBottom:4}}>/ {targetCal} kcal</span>
              </div>
              <div style={{background:'rgba(255,255,255,.07)',borderRadius:99,height:7,overflow:'hidden'}}>
                <div style={{height:'100%',borderRadius:99,width:Math.min(todayCal/targetCal*100,100)+'%',background:todayCal>targetCal?'#f87171':'linear-gradient(90deg,#818cf8,#a78bfa)',transition:'width .5s'}}/>
              </div>
              <div style={{display:'flex',justifyContent:'space-between',marginTop:6,fontSize:11,color:'rgba(255,255,255,.3)'}}>
                <span>已攝取</span><span>{Math.max(targetCal-todayCal,0)} kcal 剩餘</span>
              </div>
            </div>
            {/* macros */}
            <Card>
              <SectionTitle mb={12}>巨量營養素</SectionTitle>
              <div style={{display:'grid',gridTemplateColumns:'1fr 1fr 1fr',gap:12}}>
                <Ring value={todayP} max={Math.round(parseFloat(profile.weight)*1.6)} color='#4ade80' label='蛋白質' sub={Math.round(todayP)+'g'}/>
                <Ring value={todayC} max={Math.round(targetCal*0.45/4)} color='#facc15' label='碳水' sub={Math.round(todayC)+'g'}/>
                <Ring value={todayF} max={Math.round(targetCal*0.3/9)} color='#fb923c' label='脂肪' sub={Math.round(todayF)+'g'}/>
              </div>
            </Card>
            {/* water + weight */}
            <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:12}}>
              <div style={{background:'rgba(56,189,248,.08)',border:'1px solid rgba(56,189,248,.18)',borderRadius:20,padding:16}}>
                <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:8}}>今日飲水</div>
                <div style={{fontFamily:'DM Mono,monospace',fontSize:20,fontWeight:700,color:'#38bdf8'}}>{water}<span style={{fontSize:11}}>ml</span></div>
                <div style={{background:'rgba(255,255,255,.07)',borderRadius:99,height:4,marginTop:8,overflow:'hidden'}}>
                  <div style={{height:'100%',borderRadius:99,width:Math.min(water/profile.waterGoal*100,100)+'%',background:'#38bdf8',transition:'width .4s'}}/>
                </div>
                <div style={{fontSize:10,color:'rgba(255,255,255,.3)',marginTop:4}}>目標 {profile.waterGoal}ml</div>
              </div>
              <div style={{background:'rgba(167,139,250,.08)',border:'1px solid rgba(167,139,250,.18)',borderRadius:20,padding:16}}>
                <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:8}}>今日體重</div>
                <div style={{fontFamily:'DM Mono,monospace',fontSize:20,fontWeight:700,color:'#a78bfa'}}>{todayW?todayW.w:'—'}<span style={{fontSize:11}}>kg</span></div>
                {sortedW.length>=2&&(function(){var diff=+(sortedW[sortedW.length-1].w-sortedW[sortedW.length-2].w).toFixed(1);return <div style={{fontSize:11,color:diff>0?'#f87171':'#4ade80',marginTop:4}}>{diff>0?'+':''}{diff}kg</div>;})()}
                <div style={{fontSize:10,color:'rgba(255,255,255,.3)',marginTop:4}}>TDEE {tdee} kcal</div>
              </div>
            </div>
            {/* today meal summary */}
            {todayItems.length>0&&(
              <Card>
                <SectionTitle mb={12}>今日飲食</SectionTitle>
                {MEALS.map(function(m){
                  var items=todayDiary[m.id]||[];
                  if(!items.length)return null;
                  var mCal=items.reduce(function(s,f){return s+(f.cal||0);},0);
                  return(
                    <div key={m.id} style={{display:'flex',justifyContent:'space-between',alignItems:'center',padding:'7px 0',borderBottom:'1px solid rgba(255,255,255,.04)'}}>
                      <div style={{display:'flex',alignItems:'center',gap:8}}>
                        <span style={{fontSize:16}}>{m.icon}</span>
                        <span style={{fontSize:13}}>{m.label}</span>
                        <span style={{fontSize:11,color:'rgba(255,255,255,.3)'}}>({items.length}樣)</span>
                      </div>
                      <span style={{fontFamily:'DM Mono,monospace',fontSize:13,color:m.color}}>{mCal} kcal</span>
                    </div>
                  );
                })}
              </Card>
            )}
            {/* latest inbody */}
            {inbodyLogs.length>0&&(function(){
              var last=[...inbodyLogs].sort(function(a,b){return b.date.localeCompare(a.date)||b.id-a.id;})[0];
              return(
                <Card>
                  <SectionTitle mb={12}>最新體組成 · {last.date}</SectionTitle>
                  <div style={{display:'grid',gridTemplateColumns:'repeat(4,1fr)',gap:8}}>
                    {[{label:'體脂',val:last.bodyFat,unit:'%',color:'#f87171'},
                      {label:'肌肉',val:last.muscleMass,unit:'kg',color:'#4ade80'},
                      {label:'BMI',val:last.bmi,unit:'',color:'#facc15'},
                      {label:'內臟',val:last.visceralFat,unit:'級',color:'#fb923c'}
                    ].filter(function(x){return x.val!=null;}).map(function(x,i){
                      return(
                        <div key={i} style={{textAlign:'center'}}>
                          <div style={{fontFamily:'DM Mono,monospace',fontSize:15,color:x.color,fontWeight:700}}>{x.val}<span style={{fontSize:9}}>{x.unit}</span></div>
                          <div style={{fontSize:10,color:'rgba(255,255,255,.4)',marginTop:2}}>{x.label}</div>
                        </div>
                      );
                    })}
                  </div>
                </Card>
              );
            })()}
          </div>
        )}

        {/* ── DIET ── */}
        {tab==='diet'&&(
          <div style={{display:'flex',flexDirection:'column',gap:14}} className='fade-up'>
            {/* date nav */}
            <div style={{display:'flex',alignItems:'center',gap:10}}>
              <button onClick={function(){var d=new Date(dietDate);d.setDate(d.getDate()-1);setDietDate(d.toISOString().split('T')[0]);closePanel();}}
                style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:10,color:'white',padding:'8px 14px',fontSize:18}}>‹</button>
              <div style={{flex:1,textAlign:'center'}}>
                <input type='date' value={dietDate} onChange={function(e){setDietDate(e.target.value);closePanel();}}
                  style={{background:'transparent',border:'none',color:'white',fontSize:14,fontWeight:600,textAlign:'center',width:'100%'}}/>
                {dietDate===todayStr()&&<div style={{fontSize:10,color:'#818cf8',marginTop:1}}>今天</div>}
              </div>
              <button onClick={function(){var d=new Date(dietDate);d.setDate(d.getDate()+1);var nd=d.toISOString().split('T')[0];if(nd<=todayStr()){setDietDate(nd);closePanel();}}}
                style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:10,color:dietDate>=todayStr()?'rgba(255,255,255,.2)':'white',padding:'8px 14px',fontSize:18}}>›</button>
            </div>
            {/* day calorie bar */}
            <div style={{background:'linear-gradient(135deg,rgba(129,140,248,.18),rgba(99,102,241,.07))',border:'1px solid rgba(129,140,248,.2)',borderRadius:18,padding:'16px 18px'}}>
              <div style={{display:'flex',justifyContent:'space-between',alignItems:'flex-end',marginBottom:10}}>
                <div>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:4}}>當日熱量</div>
                  <span style={{fontFamily:'DM Mono,monospace',fontSize:30,fontWeight:700,color:'#818cf8',lineHeight:1}}>{dayCal}</span>
                  <span style={{fontSize:12,color:'rgba(255,255,255,.3)',marginLeft:6}}>/ {targetCal} kcal</span>
                </div>
                <div style={{textAlign:'right',fontSize:11,color:'rgba(255,255,255,.35)',lineHeight:1.8}}>
                  <div>P {Math.round(dayP)}g</div>
                  <div>C {Math.round(dayC)}g</div>
                  <div>F {Math.round(dayF)}g</div>
                </div>
              </div>
              <div style={{background:'rgba(255,255,255,.07)',borderRadius:99,height:6,overflow:'hidden'}}>
                <div style={{height:'100%',borderRadius:99,width:Math.min(dayCal/targetCal*100,100)+'%',background:dayCal>targetCal?'#f87171':'linear-gradient(90deg,#818cf8,#a78bfa)',transition:'width .5s'}}/>
              </div>
            </div>
            {/* meal cards */}
            {MEALS.map(function(m){
              var items=dayMeals[m.id]||[];
              var mCal=items.reduce(function(s,f){return s+(f.cal||0);},0);
              var isActive=activeMeal===m.id;
              return(
                <Card key={m.id} style={{border:'1px solid '+(isActive?m.color+'55':'rgba(255,255,255,.07)')}}>
                  {/* header */}
                  <div style={{display:'flex',alignItems:'center',marginBottom:isActive||items.length?12:0}}>
                    <span style={{fontSize:20,marginRight:10}}>{m.icon}</span>
                    <div style={{flex:1}}>
                      <div style={{fontSize:14,fontWeight:600}}>{m.label}</div>
                      {mCal>0&&<div style={{fontSize:11,color:m.color,fontFamily:'DM Mono,monospace',marginTop:1}}>{mCal} kcal</div>}
                    </div>
                    {!isActive
                      ?<div style={{display:'flex',gap:6}}>
                        <button onClick={function(){closePanel();setActiveMeal(m.id);setMealMode('text');}}
                          style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:9,color:'rgba(255,255,255,.7)',padding:'6px 10px',fontSize:12}}>✏️ 輸入</button>
                        <button onClick={function(){closePanel();setActiveMeal(m.id);setMealMode('photo');}}
                          style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:9,color:m.color,padding:'6px 10px',fontSize:12}}>📷 拍照</button>
                      </div>
                      :<button onClick={closePanel} style={{background:'none',border:'none',color:'rgba(255,255,255,.3)',fontSize:20}}>×</button>
                    }
                  </div>
                  {/* existing items */}
                  {items.map(function(f){
                    return(
                      <div key={f.id} style={{display:'flex',justifyContent:'space-between',alignItems:'center',padding:'6px 0',borderBottom:'1px solid rgba(255,255,255,.04)'}}>
                        <div style={{display:'flex',alignItems:'center',gap:8}}>
                          <div style={{width:6,height:6,borderRadius:'50%',background:(SCORE[f.score]||SCORE.good).color,flexShrink:0}}/>
                          <span style={{fontSize:12}}>{f.name}</span>
                        </div>
                        <div style={{display:'flex',alignItems:'center',gap:6}}>
                          <span style={{fontSize:11,background:(SCORE[f.score]||SCORE.good).bg,color:(SCORE[f.score]||SCORE.good).color,padding:'1px 6px',borderRadius:99}}>{(SCORE[f.score]||SCORE.good).label}</span>
                          <span style={{fontFamily:'DM Mono,monospace',fontSize:11,color:'rgba(255,255,255,.4)'}}>{f.cal}</span>
                          <button onClick={function(){removeItem(m.id,f.id);}} style={{background:'none',border:'none',color:'rgba(255,255,255,.2)',fontSize:16,lineHeight:1}}>×</button>
                        </div>
                      </div>
                    );
                  })}
                  {/* input panel */}
                  {isActive&&(
                    <div style={{marginTop:12}}>
                      {/* mode switch */}
                      <div style={{display:'flex',gap:6,marginBottom:12,background:'rgba(255,255,255,.05)',borderRadius:10,padding:4}}>
                        {[{id:'text',label:'✏️ 文字輸入'},{id:'photo',label:'📷 拍照'}].map(function(md){
                          return(
                            <button key={md.id} onClick={function(){setMealMode(md.id);setFoodResult(null);setFoodImg(null);setFoodError('');setMealText('');}}
                              style={{flex:1,background:mealMode===md.id?'rgba(255,255,255,.12)':'none',border:'none',borderRadius:8,color:mealMode===md.id?'white':'rgba(255,255,255,.4)',padding:'7px',fontSize:12,fontWeight:mealMode===md.id?600:400}}>
                              {md.label}
                            </button>
                          );
                        })}
                      </div>
                      {/* text input */}
                      {mealMode==='text'&&!foodResult&&(
                        <div>
                          <textarea value={mealText} onChange={function(e){setMealText(e.target.value);}} placeholder={'描述吃了什麼，例如：\n雞胸肉200g、白飯一碗、炒青菜'} style={{height:88,marginBottom:8}}/>
                          {foodLoading?<Spinner color={m.color} label='AI 估算中...'/>:(
                            <button onClick={handleFoodText} disabled={!mealText.trim()}
                              style={{width:'100%',background:'linear-gradient(135deg,'+m.color+','+m.color+'99)',border:'none',borderRadius:10,color:'#0b0f1a',padding:11,fontSize:13,fontWeight:700,opacity:mealText.trim()?1:0.4}}>
                              AI 分析營養 →
                            </button>
                          )}
                        </div>
                      )}
                      {/* photo input */}
                      {mealMode==='photo'&&!foodImg&&!foodResult&&<UploadZone onFile={handleFoodFile} icon='🍱' label='拍攝或上傳餐點照片'/>}
                      {foodImg&&<img src={foodImg} style={{width:'100%',borderRadius:10,maxHeight:160,objectFit:'cover',marginBottom:10}} alt=''/>}
                      {mealMode==='photo'&&foodLoading&&<Spinner color={m.color}/>}
                      {/* error */}
                      {foodError&&(
                        <div>
                          <div style={{color:'#f87171',fontSize:12,textAlign:'center',padding:'6px 0'}}>{foodError}</div>
                          <button onClick={function(){setFoodError('');setFoodImg(null);setMealText('');}} style={{width:'100%',background:'rgba(255,255,255,.06)',border:'none',borderRadius:10,color:'rgba(255,255,255,.5)',padding:8,fontSize:12,marginTop:6}}>重試</button>
                        </div>
                      )}
                      {/* result */}
                      {foodResult&&(
                        <div>
                          <div style={{background:'rgba(74,222,128,.07)',border:'1px solid rgba(74,222,128,.18)',borderRadius:10,padding:'8px 12px',marginBottom:10,fontSize:12,color:'#4ade80',lineHeight:1.5}}>
                            💡 {foodResult.advice}
                          </div>
                          {(foodResult.items||[]).map(function(item,i){
                            return(
                              <div key={i} style={{display:'flex',justifyContent:'space-between',alignItems:'center',padding:'7px 0',borderBottom:'1px solid rgba(255,255,255,.04)'}}>
                                <div>
                                  <div style={{fontSize:13}}>{item.name}</div>
                                  <div style={{fontSize:10,color:'rgba(255,255,255,.35)',marginTop:1}}>P:{item.p}g C:{item.c}g F:{item.f}g</div>
                                </div>
                                <div style={{display:'flex',alignItems:'center',gap:6}}>
                                  <span style={{fontSize:11,background:(SCORE[item.score]||SCORE.good).bg,color:(SCORE[item.score]||SCORE.good).color,padding:'1px 6px',borderRadius:99}}>{(SCORE[item.score]||SCORE.good).label}</span>
                                  <span style={{fontFamily:'DM Mono,monospace',fontSize:12,color:'#818cf8'}}>{item.cal}</span>
                                </div>
                              </div>
                            );
                          })}
                          <div style={{display:'flex',justifyContent:'space-between',fontSize:12,padding:'8px 0 0',color:'rgba(255,255,255,.5)'}}>
                            <span>合計</span>
                            <span style={{fontFamily:'DM Mono,monospace',color:'#818cf8',fontWeight:700}}>{(foodResult.total||{}).cal||0} kcal</span>
                          </div>
                          <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:8,marginTop:10}}>
                            <button onClick={function(){setFoodResult(null);setFoodImg(null);setMealText('');setFoodError('');}}
                              style={{background:'rgba(255,255,255,.07)',border:'none',borderRadius:10,color:'rgba(255,255,255,.55)',padding:10,fontSize:12}}>重新輸入</button>
                            <button onClick={confirmAdd}
                              style={{background:'linear-gradient(135deg,'+m.color+','+m.color+'99)',border:'none',borderRadius:10,color:'#0b0f1a',padding:10,fontSize:12,fontWeight:700}}>
                              加入{m.label} ✓
                            </button>
                          </div>
                        </div>
                      )}
                    </div>
                  )}
                </Card>
              );
            })}
          </div>
        )}

        {/* ── WEIGHT ── */}
        {tab==='weight'&&(
          <div style={{display:'flex',flexDirection:'column',gap:14}} className='fade-up'>
            <div style={{background:'rgba(167,139,250,.08)',border:'1px solid rgba(167,139,250,.2)',borderRadius:20,padding:20}}>
              <SectionTitle>記錄今日體重</SectionTitle>
              <div style={{display:'flex',gap:10}}>
                <input type='number' step='0.1' inputMode='decimal' value={todayWeight} onChange={function(e){setTodayWeight(e.target.value);}} placeholder='體重 (kg)'/>
                <button onClick={function(){var w=parseFloat(todayWeight);if(!w)return;setWeightLog(function(l){return l.filter(function(x){return x.d!==todayStr();}).concat([{d:todayStr(),w:w}]);});setTodayWeight('');}}
                  style={{background:'linear-gradient(135deg,#a78bfa,#8b5cf6)',border:'none',borderRadius:12,color:'white',padding:'10px 18px',fontSize:14,fontWeight:600,whiteSpace:'nowrap'}}>記錄</button>
              </div>
            </div>
            <Card>
              <SectionTitle>近期趨勢</SectionTitle>
              {sortedW.length>=2
                ?<MiniChart data={sortedW.slice(-14)} valueKey='w' color='#818cf8'/>
                :<div style={{color:'rgba(255,255,255,.3)',fontSize:13}}>需至少 2 筆記錄</div>
              }
            </Card>
            <Card>
              <SectionTitle>歷史記錄</SectionTitle>
              {[...weightLog].sort(function(a,b){return b.d.localeCompare(a.d);}).slice(0,20).map(function(e,i){
                return(
                  <div key={i} style={{display:'flex',justifyContent:'space-between',padding:'8px 0',borderBottom:'1px solid rgba(255,255,255,.04)',fontSize:13}}>
                    <span style={{color:'rgba(255,255,255,.5)'}}>{e.d}</span>
                    <div style={{display:'flex',gap:10,alignItems:'center'}}>
                      <span style={{fontFamily:'DM Mono,monospace',color:'#a78bfa'}}>{e.w} kg</span>
                      <button onClick={function(){setWeightLog(function(l){return l.filter(function(x){return x.d!==e.d;});});}} style={{background:'none',border:'none',color:'rgba(255,255,255,.15)',fontSize:14}}>×</button>
                    </div>
                  </div>
                );
              })}
              {!weightLog.length&&<div style={{color:'rgba(255,255,255,.3)',fontSize:13}}>尚無記錄</div>}
            </Card>
          </div>
        )}

        {/* ── WATER ── */}
        {tab==='water'&&(
          <div style={{display:'flex',flexDirection:'column',gap:14}} className='fade-up'>
            <div style={{background:'rgba(56,189,248,.08)',border:'1px solid rgba(56,189,248,.2)',borderRadius:20,padding:28,textAlign:'center'}}>
              <div style={{fontFamily:'DM Serif Display,serif',fontSize:56,color:'#38bdf8',lineHeight:1}}>{water}</div>
              <div style={{fontSize:13,color:'rgba(255,255,255,.4)',marginTop:6}}>ml / 目標 {profile.waterGoal} ml</div>
              <div style={{background:'rgba(255,255,255,.07)',borderRadius:99,height:10,margin:'18px 0 8px',overflow:'hidden'}}>
                <div style={{height:'100%',borderRadius:99,width:Math.min(water/profile.waterGoal*100,100)+'%',background:'linear-gradient(90deg,#38bdf8,#0ea5e9)',transition:'width .4s'}}/>
              </div>
              <div style={{fontSize:12,color:'rgba(255,255,255,.4)'}}>{Math.round(water/profile.waterGoal*100)}%</div>
            </div>
            <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:10}}>
              {[150,200,250,500].map(function(ml){
                return(
                  <button key={ml} onClick={function(){setWater(function(w){return Math.min(w+ml,profile.waterGoal*1.5);});}}
                    style={{background:'rgba(56,189,248,.1)',border:'1px solid rgba(56,189,248,.2)',borderRadius:16,color:'#38bdf8',padding:18,fontSize:17,fontWeight:600,fontFamily:'DM Mono,monospace'}}>
                    +{ml}ml
                  </button>
                );
              })}
            </div>
            <button onClick={function(){setWater(0);}} style={{background:'rgba(255,255,255,.04)',border:'1px solid rgba(255,255,255,.08)',borderRadius:16,color:'rgba(255,255,255,.4)',padding:13,fontSize:13}}>重置</button>
          </div>
        )}

        {/* ── PROFILE ── */}
        {tab==='profile'&&(
          <div style={{display:'flex',flexDirection:'column',gap:14}} className='fade-up'>
            {/* inbody subtabs */}
            <div style={{display:'flex',gap:6,background:'rgba(255,255,255,.04)',borderRadius:14,padding:4}}>
              {[{id:'scan',label:'掃描報告'},{id:'history',label:'歷史記錄'+(inbodyLogs.length?' ('+inbodyLogs.length+')':'')}].map(function(t){
                return(
                  <button key={t.id} onClick={function(){setInbodyTab(t.id);}}
                    style={{flex:1,background:inbodyTab===t.id?'rgba(129,140,248,.25)':'none',border:'none',borderRadius:10,color:inbodyTab===t.id?'white':'rgba(255,255,255,.4)',padding:8,fontSize:13,fontWeight:inbodyTab===t.id?600:400}}>
                    {t.label}
                  </button>
                );
              })}
            </div>
            {/* scan */}
            {inbodyTab==='scan'&&(
              <Card>
                <div style={{display:'flex',alignItems:'center',gap:8,marginBottom:14}}>
                  <div style={{width:6,height:6,borderRadius:'50%',background:'#818cf8'}}/>
                  <div style={{fontSize:12,color:'rgba(255,255,255,.5)'}}>InBody 報告掃描</div>
                  <div style={{marginLeft:'auto',fontSize:11,background:'rgba(129,140,248,.12)',color:'#818cf8',padding:'2px 8px',borderRadius:99}}>Claude Vision</div>
                </div>
                {!ibImg
                  ?<UploadZone onFile={function(f){setIbDate(todayStr());handleIbFile(f);}} icon='📋' label='拍攝 InBody 報告，AI 自動讀取數值'/>
                  :<div>
                    <img src={ibImg} style={{width:'100%',borderRadius:12,maxHeight:160,objectFit:'cover',marginBottom:12}} alt=''/>
                    {ibLoading&&<Spinner color='#818cf8'/>}
                    {ibError&&<div style={{color:'#f87171',fontSize:13,textAlign:'center'}}>{ibError}</div>}
                    {ibResult&&(
                      <div>
                        {ibResult.assessment&&<div style={{background:'rgba(129,140,248,.08)',border:'1px solid rgba(129,140,248,.2)',borderRadius:12,padding:'10px 14px',marginBottom:12,fontSize:13,color:'#818cf8',lineHeight:1.6}}>🔍 {ibResult.assessment}</div>}
                        <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:8,marginBottom:12}}>
                          {[
                            {label:'體重',val:ibResult.weight,unit:'kg'},
                            {label:'體脂率',val:ibResult.bodyFat,unit:'%'},
                            {label:'骨骼肌',val:ibResult.muscleMass,unit:'kg'},
                            {label:'體脂肪',val:ibResult.fatMass,unit:'kg'},
                            {label:'BMI',val:ibResult.bmi,unit:''},
                            {label:'基礎代謝',val:ibResult.bmr,unit:'kcal'},
                            {label:'內臟脂肪',val:ibResult.visceralFat,unit:'級'},
                            {label:'體水分',val:ibResult.bodyWater,unit:'kg'}
                          ].filter(function(x){return x.val!=null;}).map(function(x,i){
                            return(
                              <div key={i} style={{background:'rgba(255,255,255,.04)',borderRadius:10,padding:'10px 12px'}}>
                                <div style={{fontSize:11,color:'rgba(255,255,255,.4)'}}>{x.label}</div>
                                <div style={{fontFamily:'DM Mono,monospace',fontSize:16,color:'#818cf8',fontWeight:700}}>{x.val}<span style={{fontSize:11}}>{x.unit}</span></div>
                              </div>
                            );
                          })}
                        </div>
                        {ibResult.suggestions&&ibResult.suggestions.map(function(s,i){
                          return <div key={i} style={{display:'flex',gap:8,fontSize:12,color:'rgba(255,255,255,.55)',padding:'3px 0'}}><span style={{color:'#818cf8'}}>→</span><span>{s}</span></div>;
                        })}
                        {/* date picker */}
                        <div style={{background:'rgba(255,255,255,.05)',borderRadius:12,padding:'12px 14px',margin:'12px 0'}}>
                          <div style={{fontSize:11,color:'rgba(255,255,255,.45)',marginBottom:6}}>量測日期</div>
                          <input type='date' value={ibDate} onChange={function(e){setIbDate(e.target.value);}}/>
                          {inbodyLogs.filter(function(x){return x.date===ibDate;}).length>0&&(
                            <div style={{fontSize:11,color:'#facc15',marginTop:6}}>
                              📅 此日期已有 {inbodyLogs.filter(function(x){return x.date===ibDate;}).length} 筆，將新增第 {inbodyLogs.filter(function(x){return x.date===ibDate;}).length+1} 筆
                            </div>
                          )}
                        </div>
                        <div style={{display:'grid',gridTemplateColumns:'1fr 1fr',gap:8}}>
                          <button onClick={function(){setIbImg(null);setIbResult(null);}} style={{background:'rgba(255,255,255,.06)',border:'none',borderRadius:12,color:'rgba(255,255,255,.5)',padding:11,fontSize:13}}>重新上傳</button>
                          <button onClick={saveIbLog} style={{background:'linear-gradient(135deg,#818cf8,#6366f1)',border:'none',borderRadius:12,color:'white',padding:11,fontSize:13,fontWeight:700}}>儲存記錄 ✓</button>
                        </div>
                      </div>
                    )}
                    {!ibLoading&&!ibResult&&!ibError&&(
                      <button onClick={function(){setIbImg(null);}} style={{background:'rgba(255,255,255,.05)',border:'none',borderRadius:12,color:'rgba(255,255,255,.4)',padding:10,fontSize:13,width:'100%'}}>取消</button>
                    )}
                  </div>
                }
              </Card>
            )}
            {inbodyTab==='history'&&<InBodyHistory logs={inbodyLogs} onDelete={function(id){setInbodyLogs(function(l){return l.filter(function(x){return x.id!==id;});});}}/>}
            {/* manual profile */}
            <Card>
              <SectionTitle>基本資料</SectionTitle>
              <div style={{display:'flex',flexDirection:'column',gap:10}}>
                {[
                  {key:'name',label:'姓名',type:'text',ph:'你的名字'},
                  {key:'age',label:'年齡',type:'number',ph:'25'},
                  {key:'height',label:'身高 (cm)',type:'number',ph:'170'},
                  {key:'weight',label:'體重 (kg)',type:'number',ph:'70'},
                  {key:'bodyFat',label:'體脂率 (%) 可留空',type:'number',ph:'20'},
                  {key:'waterGoal',label:'每日飲水目標 (ml)',type:'number',ph:'2000'}
                ].map(function(f){
                  return(
                    <div key={f.key}>
                      <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:4}}>{f.label}</div>
                      <input type={f.type} value={profile[f.key]} placeholder={f.ph} inputMode={f.type==='number'?'decimal':'text'}
                        onChange={function(e){var v=e.target.value;setProfile(function(p){var n=Object.assign({},p);n[f.key]=v;return n;});}}/>
                    </div>
                  );
                })}
                <div>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:4}}>性別</div>
                  <select value={profile.gender} onChange={function(e){var v=e.target.value;setProfile(function(p){return Object.assign({},p,{gender:v});});}}>
                    <option value='male'>男</option><option value='female'>女</option>
                  </select>
                </div>
                <div>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:4}}>活動量</div>
                  <select value={profile.activity} onChange={function(e){var v=e.target.value;setProfile(function(p){return Object.assign({},p,{activity:v});});}}>
                    {Object.entries(ACT).map(function(kv){return <option key={kv[0]} value={kv[0]}>{kv[1]}</option>;})}
                  </select>
                </div>
                <div>
                  <div style={{fontSize:11,color:'rgba(255,255,255,.4)',marginBottom:4}}>目標</div>
                  <select value={profile.goal} onChange={function(e){var v=e.target.value;setProfile(function(p){return Object.assign({},p,{goal:v});});}}>
                    <option value='lose'>減重 (−500 kcal)</option>
                    <option value='maintain'>維持體重</option>
                    <option value='gain'>增肌 (+300 kcal)</option>
                  </select>
                </div>
                <button onClick={function(){setProfileSaved(true);setTimeout(function(){setProfileSaved(false);},2000);}}
                  style={{background:'linear-gradient(135deg,#818cf8,#6366f1)',border:'none',borderRadius:12,color:'white',padding:14,fontSize:14,fontWeight:600,marginTop:4}}>
                  {profileSaved?'✓ 已儲存':'儲存資料'}
                </button>
              </div>
            </Card>
            <Card>
              <SectionTitle>計算結果</SectionTitle>
              {[
                {label:'基礎代謝率 (BMR)',val:bmr+' kcal',color:'#818cf8'},
                {label:'總熱量消耗 (TDEE)',val:tdee+' kcal',color:'#a78bfa'},
                {label:'每日攝取目標',val:targetCal+' kcal',color:'#4ade80'},
                {label:'建議蛋白質',val:Math.round(parseFloat(profile.weight)*1.6)+'g',color:'#34d399'},
                {label:'BMI',val:(parseFloat(profile.weight)/(parseFloat(profile.height)/100)**2).toFixed(1),color:'#facc15'}
              ].map(function(s){
                return(
                  <div key={s.label} style={{display:'flex',justifyContent:'space-between',padding:'10px 0',borderBottom:'1px solid rgba(255,255,255,.04)'}}>
                    <span style={{fontSize:13,color:'rgba(255,255,255,.55)'}}>{s.label}</span>
                    <span style={{fontFamily:'DM Mono,monospace',fontSize:13,color:s.color,fontWeight:700}}>{s.val}</span>
                  </div>
                );
              })}
            </Card>
            <button onClick={function(){LS.set('bl_apikey','');setApiKey('');}}
              style={{background:'rgba(248,113,113,.08)',border:'1px solid rgba(248,113,113,.2)',borderRadius:12,color:'#f87171',padding:12,fontSize:13,marginBottom:8}}>
              更換 API Key
            </button>
          </div>
        )}
      </div>

      {/* bottom nav */}
      <div style={{position:'absolute',bottom:0,left:0,right:0,background:'rgba(11,15,26,.95)',backdropFilter:'blur(24px)',WebkitBackdropFilter:'blur(24px)',borderTop:'1px solid rgba(255,255,255,.07)',display:'flex',paddingBottom:'env(safe-area-inset-bottom)',zIndex:100}}>
        {TABS.map(function(t){
          return(
            <button key={t.id} onClick={function(){setTab(t.id);}}
              style={{flex:1,background:'none',border:'none',color:tab===t.id?'#818cf8':'rgba(255,255,255,.28)',display:'flex',flexDirection:'column',alignItems:'center',gap:3,padding:'10px 0'}}>
              <span style={{fontSize:20,lineHeight:1}}>{t.icon}</span>
              <span style={{fontSize:10,fontWeight:tab===t.id?600:400}}>{t.label}</span>
            </button>
          );
        })}
      </div>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(React.createElement(App));
</script>
</body>
</html>
