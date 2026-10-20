/\*\*  
 \* BGPCommunityForge Pro \- Studio UI Controller  
 \*/

// Global State  
let currentPolicy \= {  
  localAsn: 65001,  
  peerType: 'UPSTREAM\_TRANSIT',  
  targetProviderId: 'lumen',  
  targetAsn: 3356,  
  peerIp: '192.0.2.1',  
  prefixes: \['198.51.100.0/24'\],  
  allowTransitReExport: false,  
  otcEnforced: true,  
  asPathPrepend: 1,  
  communities: \['3356:100', 'NO\_EXPORT'\],  
  largeCommunities: \['65001:100:1'\],  
  stripPrivateAsn: true,  
  enableGracefulShutdown: false,  
  isRtbhBlackhole: false  
};

let currentVendorTab \= 'cisco'; // 'cisco', 'juniper', 'arista', 'bird', 'frr'

const PRESETS \= {  
  dual\_homed: {  
    localAsn: 65001,  
    peerType: 'UPSTREAM\_TRANSIT',  
    targetProviderId: 'lumen',  
    targetAsn: 3356,  
    peerIp: '192.0.2.1',  
    prefixes: \['198.51.100.0/24'\],  
    allowTransitReExport: false,  
    otcEnforced: true,  
    asPathPrepend: 1,  
    communities: \['3356:100', 'NO\_EXPORT'\],  
    largeCommunities: \['65001:100:1'\],  
    stripPrivateAsn: true,  
    enableGracefulShutdown: false,  
    isRtbhBlackhole: false  
  },  
  route\_leak: {  
    localAsn: 65001,  
    peerType: 'UPSTREAM\_TRANSIT',  
    targetProviderId: 'cogent',  
    targetAsn: 174,  
    peerIp: '198.18.0.1',  
    prefixes: \['198.51.100.0/24', '8.8.8.0/24'\],  
    allowTransitReExport: true, // CATASTROPHIC LEAK\!  
    otcEnforced: false,  
    asPathPrepend: 0,  
    communities: \['174:100'\],  
    largeCommunities: \[\],  
    stripPrivateAsn: true,  
    enableGracefulShutdown: false,  
    isRtbhBlackhole: false  
  },  
  rtbh: {  
    localAsn: 65001,  
    peerType: 'UPSTREAM\_TRANSIT',  
    targetProviderId: 'lumen',  
    targetAsn: 3356,  
    peerIp: '192.0.2.1',  
    prefixes: \['198.51.100.55/32'\],  
    allowTransitReExport: false,  
    otcEnforced: true,  
    asPathPrepend: 0,  
    communities: \['65535:666', '3356:9999', 'NO\_EXPORT'\],  
    largeCommunities: \['65001:666:1'\],  
    stripPrivateAsn: true,  
    enableGracefulShutdown: false,  
    isRtbhBlackhole: true  
  }  
};

document.addEventListener('DOMContentLoaded', () \=\> {  
  loadPreset('dual\_homed');  
});

function loadPreset(key) {  
  const p \= PRESETS\[key\];  
  if (\!p) return;

  \['btnPresetDualHomed', 'btnPresetLeak', 'btnPresetRtbh'\].forEach(id \=\> {  
    document.getElementById(id)?.classList.remove('active');  
  });

  if (key \=== 'dual\_homed') document.getElementById('btnPresetDualHomed')?.classList.add('active');  
  if (key \=== 'route\_leak') document.getElementById('btnPresetLeak')?.classList.add('active');  
  if (key \=== 'rtbh') document.getElementById('btnPresetRtbh')?.classList.add('active');

  currentPolicy \= JSON.parse(JSON.stringify(p));

  // Populate UI inputs  
  document.getElementById('txtLocalAsn').value \= currentPolicy.localAsn;  
  document.getElementById('selPeerType').value \= currentPolicy.peerType;  
  document.getElementById('selTargetProvider').value \= currentPolicy.targetProviderId;  
  document.getElementById('txtPeerIp').value \= currentPolicy.peerIp;  
  document.getElementById('txtPrefixes').value \= (currentPolicy.prefixes || \[\]).join('\\n');  
  document.getElementById('chkAllowTransitReExport').checked \= \!\!currentPolicy.allowTransitReExport;  
  document.getElementById('selPrepend').value \= String(currentPolicy.asPathPrepend || 0);  
  document.getElementById('txtCommunities').value \= (currentPolicy.communities || \[\]).join(', ');  
  document.getElementById('txtLargeCommunities').value \= (currentPolicy.largeCommunities || \[\]).join(', ');  
  document.getElementById('chkStripPrivateAsn').checked \= currentPolicy.stripPrivateAsn \!== false;  
  document.getElementById('chkOtcEnforced').checked \= currentPolicy.otcEnforced \!== false;  
  document.getElementById('chkGracefulShutdown').checked \= \!\!currentPolicy.enableGracefulShutdown;  
  document.getElementById('chkRtbhBlackhole').checked \= \!\!currentPolicy.isRtbhBlackhole;

  updateProviderCommunitiesDropdown();  
  runLiveAudit();  
}

function updateProviderCommunitiesDropdown() {  
  const selProv \= document.getElementById('selTargetProvider').value;  
  const prov \= BGPProviderDatabase.findById(selProv);  
  const selComm \= document.getElementById('selProviderCommunity');  
  selComm.innerHTML \= '';

  if (\!prov) {  
    selComm.innerHTML \= '\<option value=""\>No provider communities\</option\>';  
    return;  
  }

  const optGroupLp \= document.createElement('optgroup');  
  optGroupLp.label \= \`\${prov.name} \- Local Preference\`;  
  (prov.localPref || \[\]).forEach(lp \=\> {  
    const opt \= document.createElement('option');  
    opt.value \= lp.community;  
    opt.innerText \= \`\${lp.community} (\${lp.label})\`;  
    optGroupLp.appendChild(opt);  
  });  
  selComm.appendChild(optGroupLp);

  const optGroupPrep \= document.createElement('optgroup');  
  optGroupPrep.label \= \`\${prov.name} \- AS Prepending\`;  
  (prov.prepending || \[\]).forEach(pr \=\> {  
    const opt \= document.createElement('option');  
    opt.value \= pr.community;  
    opt.innerText \= \`\${pr.community} (\${pr.label})\`;  
    optGroupPrep.appendChild(opt);  
  });  
  selComm.appendChild(optGroupPrep);

  if (prov.scopeControl && prov.scopeControl.length \> 0\) {  
    const optGroupSc \= document.createElement('optgroup');  
    optGroupSc.label \= \`\${prov.name} \- Scope Containment\`;  
    prov.scopeControl.forEach(sc \=\> {  
      const opt \= document.createElement('option');  
      opt.value \= sc.community;  
      opt.innerText \= \`\${sc.community} (\${sc.label})\`;  
      optGroupSc.appendChild(opt);  
    });  
    selComm.appendChild(optGroupSc);  
  }

  if (prov.blackhole) {  
    const optGroupBh \= document.createElement('optgroup');  
    optGroupBh.label \= \`\${prov.name} \- DDoS Blackhole\`;  
    const opt \= document.createElement('option');  
    opt.value \= prov.blackhole;  
    opt.innerText \= \`\${prov.blackhole} (RTBH Blackhole)\`;  
    optGroupBh.appendChild(opt);  
    selComm.appendChild(optGroupBh);  
  }  
}

function onProviderSelectChange() {  
  const selProv \= document.getElementById('selTargetProvider').value;  
  const prov \= BGPProviderDatabase.findById(selProv);  
  if (prov) {  
    currentPolicy.targetProviderId \= prov.id;  
    currentPolicy.targetAsn \= prov.asn;  
  }  
  updateProviderCommunitiesDropdown();  
  onParamChange();  
}

function onProviderCommunitySelect() {  
  const chosen \= document.getElementById('selProviderCommunity').value;  
  if (\!chosen) return;

  const currentList \= document.getElementById('txtCommunities').value  
    .split(/\[,\]+/)  
    .map(s \=\> s.trim())  
    .filter(Boolean);

  if (\!currentList.includes(chosen)) {  
    currentList.push(chosen);  
    document.getElementById('txtCommunities').value \= currentList.join(', ');  
  }  
  onParamChange();  
}

function onParamChange() {  
  currentPolicy.localAsn \= parseInt(document.getElementById('txtLocalAsn').value, 10\) || 65001;  
  currentPolicy.peerType \= document.getElementById('selPeerType').value;  
  currentPolicy.targetProviderId \= document.getElementById('selTargetProvider').value;  
  const prov \= BGPProviderDatabase.findById(currentPolicy.targetProviderId);  
  currentPolicy.targetAsn \= prov ? prov.asn : 3356;  
  currentPolicy.peerIp \= document.getElementById('txtPeerIp').value.trim() || '192.0.2.1';

  currentPolicy.prefixes \= document.getElementById('txtPrefixes').value  
    .split(/\[\\r\\n,\]+/)  
    .map(s \=\> s.trim())  
    .filter(Boolean);

  currentPolicy.allowTransitReExport \= document.getElementById('chkAllowTransitReExport').checked;  
  currentPolicy.asPathPrepend \= parseInt(document.getElementById('selPrepend').value, 10\) || 0;

  currentPolicy.communities \= document.getElementById('txtCommunities').value  
    .split(/\[,\]+/)  
    .map(s \=\> s.trim())  
    .filter(Boolean);

  currentPolicy.largeCommunities \= document.getElementById('txtLargeCommunities').value  
    .split(/\[,\]+/)  
    .map(s \=\> s.trim())  
    .filter(Boolean);

  currentPolicy.stripPrivateAsn \= document.getElementById('chkStripPrivateAsn').checked;  
  currentPolicy.otcEnforced \= document.getElementById('chkOtcEnforced').checked;  
  currentPolicy.enableGracefulShutdown \= document.getElementById('chkGracefulShutdown').checked;  
  currentPolicy.isRtbhBlackhole \= document.getElementById('chkRtbhBlackhole').checked;

  runLiveAudit();  
}

function runLiveAudit() {  
  const auditReport \= BGPLeakAuditor.audit(currentPolicy);

  // Update Metrics  
  document.getElementById('metricLocalAsn').innerText \= \`AS\${currentPolicy.localAsn}\`;  
  const prov \= BGPProviderDatabase.findById(currentPolicy.targetProviderId);  
  document.getElementById('metricProvider').innerText \= prov ? \`\${prov.name} (AS\${prov.asn})\` : \`AS\${currentPolicy.targetAsn}\`;

  const leakShieldEl \= document.getElementById('metricLeakShield');  
  if (auditReport.isDangerousTransitLeak) {  
    leakShieldEl.innerText \= 'BROKEN / LEAKING';  
    leakShieldEl.style.color \= 'var(--accent-red)';  
  } else {  
    leakShieldEl.innerText \= 'Active (Protected)';  
    leakShieldEl.style.color \= 'var(--accent-green)';  
  }

  // Posture Badge  
  const postureBadge \= document.getElementById('postureBadge');  
  postureBadge.innerText \= auditReport.posture.replace(/\_/g, ' ');  
  if (auditReport.posture \=== 'ZERO\_TRUST\_VERIFIED') {  
    postureBadge.className \= 'badge badge-green';  
  } else if (auditReport.posture \=== 'CATASTROPHIC\_ROUTE\_LEAK\_DISASTER') {  
    postureBadge.className \= 'badge badge-red';  
  } else {  
    postureBadge.className \= 'badge badge-amber';  
  }

  // Alert Banner  
  const banner \= document.getElementById('leakBanner');  
  if (auditReport.isDangerousTransitLeak) {  
    banner.style.display \= 'flex';  
  } else {  
    banner.style.display \= 'none';  
  }

  // Scorecard Header  
  const gradeBadge \= document.getElementById('gradeBadge');  
  gradeBadge.innerText \= auditReport.grade;  
  gradeBadge.className \= 'score-badge-huge grade-' \+ auditReport.grade.toLowerCase().replace('+', 'plus');

  document.getElementById('scoreSummary').innerText \= \`Score: \${auditReport.score} / 100\`;  
  document.getElementById('gradeSubtitle').innerText \= auditReport.isProductionReady  
    ? 'Zero-Trust Verified • Route Leak Shield Active'  
    : 'Route Leak Misconfiguration or Subnet Boundary Violation Detected';

  const prodBadge \= document.getElementById('prodReadyBadge');  
  if (auditReport.isProductionReady) {  
    prodBadge.innerText \= 'ZERO-TRUST VERIFIED';  
    prodBadge.className \= 'badge badge-green';  
  } else {  
    prodBadge.innerText \= 'UNSAFE BGP POLICY';  
    prodBadge.className \= 'badge badge-red';  
  }

  // Render Checks  
  const checksList \= document.getElementById('checksList');  
  checksList.innerHTML \= '';  
  auditReport.checks.forEach(chk \=\> {  
    const item \= document.createElement('div');  
    item.className \= 'check-item';

    const header \= document.createElement('div');  
    header.className \= 'check-header';

    const title \= document.createElement('div');  
    title.className \= 'check-title';  
    title.innerHTML \= \`\<span\>\${chk.passed ? '✅' : (chk.level \=== 'CRITICAL' ? '🛑' : '⚠️')}\</span\> \<span\>\${chk.title}\</span\>\`;

    const badge \= document.createElement('span');  
    badge.className \= \`badge \${chk.passed ? 'badge-green' : (chk.level \=== 'CRITICAL' ? 'badge-red' : 'badge-amber')}\`;  
    badge.innerText \= \`\${chk.points}/\${chk.maxPoints} pts\`;

    header.appendChild(title);  
    header.appendChild(badge);

    const msg \= document.createElement('div');  
    msg.className \= 'check-msg';  
    msg.innerText \= chk.message;

    item.appendChild(header);  
    item.appendChild(msg);

    if (\!chk.passed && chk.remediation) {  
      const rem \= document.createElement('div');  
      rem.className \= 'check-remediation';  
      rem.innerText \= '🔧 Mitigation: ' \+ chk.remediation;  
      item.appendChild(rem);  
    }

    checksList.appendChild(item);  
  });

  renderVendorSynthesizer();  
}

function switchVendorTab(tab) {  
  currentVendorTab \= tab;  
  \['tabCisco', 'tabJuniper', 'tabArista', 'tabBird', 'tabFrr'\].forEach(id \=\> {  
    document.getElementById(id)?.classList.remove('active');  
  });

  if (tab \=== 'cisco') document.getElementById('tabCisco')?.classList.add('active');  
  if (tab \=== 'juniper') document.getElementById('tabJuniper')?.classList.add('active');  
  if (tab \=== 'arista') document.getElementById('tabArista')?.classList.add('active');  
  if (tab \=== 'bird') document.getElementById('tabBird')?.classList.add('active');  
  if (tab \=== 'frr') document.getElementById('tabFrr')?.classList.add('active');

  renderVendorSynthesizer();  
}

function renderVendorSynthesizer() {  
  const codeEl \= document.getElementById('vendorConfigOutput');  
  const code \= BGPVendorSynthesizer.synthesize(currentVendorTab, currentPolicy);  
  codeEl.innerText \= code;  
}

function copyTextFromElement(elementId, alertId) {  
  const el \= document.getElementById(elementId);  
  if (\!el) return;  
  navigator.clipboard.writeText(el.innerText).then(() \=\> {  
    showAlert(alertId);  
  }).catch(() \=\> {  
    // Fallback  
    const range \= document.createRange();  
    range.selectNodeContents(el);  
    const sel \= window.getSelection();  
    sel.removeAllRanges();  
    sel.addRange(range);  
    document.execCommand('copy');  
    showAlert(alertId);  
  });  
}

function showAlert(alertId) {  
  const alertEl \= document.getElementById(alertId);  
  if (alertEl) {  
    alertEl.style.display \= 'inline';  
    setTimeout(() \=\> {  
      alertEl.style.display \= 'none';  
    }, 2000);  
  }  
}  
