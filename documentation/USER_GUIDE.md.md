\# BGPCommunityForge Pro \- User Guide & Integration Manual

\#\# 1\. Quick Start

\#\#\# Web Studio (100% Client-Side)  
1\. Open \`source-code/index.html\` in any web browser.  
2\. Select an infrastructure preset:  
   \- \*\*Dual-Homed Lumen/Cogent (A+)\*\*: Primary link via Lumen (Local-Pref 100), secondary link via Cogent (Local-Pref 70, Prepend 2x) with strict route leak prevention.  
   \- \*\*Catastrophic Route Leak (F)\*\*: Hazardous transit-to-transit route leak triggering immediate alert.  
   \- \*\*DDoS RTBH Blackhole (A+)\*\*: Emergency /32 host null-routing via RFC 7999 community \`65535:666\`.  
3\. Customize local ASN, advertised CIDR prefixes, and BGP peer type.  
4\. If an unsafe transit re-export is enabled, review the red \*\*CATASTROPHIC ROUTE LEAK DETECTED\*\* banner.  
5\. Review the \*\*10-Point Zero-Trust Scorecard\*\* to inspect which criteria passed or failed.  
6\. Click any tab in the \*\*Multi-Vendor Policy Synthesizer\*\* (Cisco, Juniper, Arista, BIRD, FRR) and click \*\*Copy Routing Configuration\*\* to deploy to your core edge routers.

\---

\#\# 2\. CLI Tool Usage (\`bgpcommunity-cli.py\`)

The standalone Python CLI requires zero external dependencies and runs on Python 3.8+.

\#\#\# 2.1 Auditing a BGP Policy with Strict NetDevOps CI/CD Gating  
\`\`\`bash  
\# Legitimate multi-homed enterprise policy \-\> Exits code 0  
python3 build/bgpcommunity-cli.py audit \--preset presets/dual-homed-enterprise-lumen-cogent.json \--strict

\# Catastrophic route leak \-\> Exits code 1 (Pipeline Fails)  
python3 build/bgpcommunity-cli.py audit \--preset presets/catastrophic-route-leak-transit.json \--strict  
\`\`\`

\#\#\# 2.2 Synthesizing Multi-Vendor Router Configurations  
\`\`\`bash  
\# Generate Cisco IOS-XE configuration  
python3 build/bgpcommunity-cli.py synthesize \--preset presets/dual-homed-enterprise-lumen-cogent.json \-v cisco

\# Generate Juniper JunOS configuration  
python3 build/bgpcommunity-cli.py synthesize \--preset presets/dual-homed-enterprise-lumen-cogent.json \-v juniper

\# Generate BIRD 2.x configuration  
python3 build/bgpcommunity-cli.py synthesize \--preset presets/dual-homed-enterprise-lumen-cogent.json \-v bird  
\`\`\`

\#\#\# 2.3 Looking Up BGP Communities  
\`\`\`bash  
\# Look up RFC 1997 well-known community  
python3 build/bgpcommunity-cli.py lookup \--community "65535:65281"  
\# Output: {"name": "NO\_EXPORT", "rfc": "RFC 1997", ...}

\# Look up RFC 7999 blackhole community  
python3 build/bgpcommunity-cli.py lookup \--community "65535:666"  
\`\`\`

\---

\#\# 3\. NetDevOps CI/CD Pipeline (GitHub Actions)

Add this workflow to your network infrastructure as code (IaC) repository to prevent route leaks before configurations are pushed to production routers:

\`\`\`yaml  
name: BGP Policy & Route Leak CI/CD Gate

on:  
  pull\_request:  
    paths:  
      \- 'bgp-policies/\*\*'  
      \- 'routers/\*\*'

jobs:  
  bgp-route-leak-audit:  
    runs-on: ubuntu-latest  
    steps:  
      \- uses: actions/checkout@v4  
      \- name: Set up Python  
        uses: actions/setup-python@v5  
        with:  
          python-version: '3.11'  
      \- name: Audit BGP Policies Against Route Leak Shield  
        run: |  
          python3 build/bgpcommunity-cli.py audit \--preset presets/dual-homed-enterprise-lumen-cogent.json \--strict  
\`\`\`  
