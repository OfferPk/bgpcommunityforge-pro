/\*\*  
 \* BGPCommunityForge Pro \- Multi-Vendor BGP Policy Synthesizer  
 \* Generates ready-to-deploy routing configurations for:  
 \* Cisco IOS-XE/XR, Juniper JunOS, Arista EOS, BIRD 2.x, and FRRouting (FRR).  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.BGPVendorSynthesizer \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  const BGPVendorSynthesizer \= {  
    /\*\*  
     \* Synthesizes policy based on vendor  
     \*/  
    synthesize(vendor, policy) {  
      const p \= policy || {};  
      const v \= (vendor || 'cisco').toLowerCase();

      switch (v) {  
        case 'cisco':  
          return this.generateCisco(p);  
        case 'juniper':  
          return this.generateJuniper(p);  
        case 'arista':  
          return this.generateArista(p);  
        case 'bird':  
          return this.generateBird(p);  
        case 'frr':  
          return this.generateFRR(p);  
        default:  
          return this.generateCisco(p);  
      }  
    },

    generateCisco(p) {  
      const asn \= p.localAsn || 65001;  
      const peerAsn \= p.targetAsn || 3356;  
      const peerIp \= p.peerIp || '192.0.2.1';  
      const prefixes \= p.prefixes || \['198.51.100.0/24'\];  
      const prepends \= parseInt(p.asPathPrepend, 10\) || 0;  
      const comms \= (p.communities || \[\]).map(c \=\> c.toLowerCase() \=== 'no\_export' ? 'no-export' : c);  
      const largeComms \= p.largeCommunities || \[\];

      let prefixLines \= '';  
      prefixes.forEach((pref, idx) \=\> {  
        prefixLines \+= \`ip prefix-list PL-BGP-OUT seq \${(idx \+ 1\) \* 10} permit \${pref}\\n\`;  
      });

      let prependClause \= '';  
      if (prepends \> 0\) {  
        const prepStr \= Array(prepends).fill(asn).join(' ');  
        prependClause \= \` set as-path prepend \${prepStr}\\n\`;  
      }

      let commClause \= '';  
      if (comms.length \> 0\) {  
        commClause \+= \` set community \${comms.join(' ')} additive\\n\`;  
      }  
      if (largeComms.length \> 0\) {  
        commClause \+= \` set large-community \${largeComms.join(' ')} additive\\n\`;  
      }

      return \`\! \========================================================  
\! Cisco IOS-XE / IOS-XR BGP Export Policy  
\! \========================================================  
\${prefixLines}\!  
route-map RM-UPSTREAM-OUT permit 10  
 description Zero-Trust Prefix Export & Traffic Engineering  
 match ip address prefix-list PL-BGP-OUT  
\${commClause}\${prependClause}\!  
route-map RM-UPSTREAM-OUT deny 99  
 description Implicit Reject \- Route Leak Shield  
\!  
router bgp \${asn}  
 neighbor \${peerIp} remote-as \${peerAsn}  
 neighbor \${peerIp} description UPSTREAM-TRANSIT-AS\${peerAsn}  
 neighbor \${peerIp} route-map RM-UPSTREAM-OUT out  
 neighbor \${peerIp} send-community both  
\${p.stripPrivateAsn \!== false ? \` neighbor \${peerIp} remove-private-as\\n\` : ''}\`;  
    },

    generateJuniper(p) {  
      const asn \= p.localAsn || 65001;  
      const peerAsn \= p.targetAsn || 3356;  
      const peerIp \= p.peerIp || '192.0.2.1';  
      const prefixes \= p.prefixes || \['198.51.100.0/24'\];  
      const prepends \= parseInt(p.asPathPrepend, 10\) || 0;  
      const comms \= p.communities || \[\];  
      const largeComms \= p.largeCommunities || \[\];

      let lines \= \`/\* \======================================================== \*/  
/\* Juniper JunOS BGP Policy-Options & Routing-Options       \*/  
/\* \======================================================== \*/\\n\`;

      prefixes.forEach(pref \=\> {  
        lines \+= \`set policy-options prefix-list PL-BGP-OUT \${pref}\\n\`;  
      });

      comms.forEach((c, idx) \=\> {  
        const commVal \= c.toLowerCase() \=== 'no\_export' ? 'no-export' : c;  
        lines \+= \`set policy-options community COMM-POLICY-\${idx \+ 1} members \${commVal}\\n\`;  
      });

      largeComms.forEach((lc, idx) \=\> {  
        lines \+= \`set policy-options community COMM-LARGE-\${idx \+ 1} members \${lc}\\n\`;  
      });

      lines \+= \`\\nset policy-options policy-statement POL-UPSTREAM-OUT term TERM-PERMIT from prefix-list PL-BGP-OUT\\n\`;

      comms.forEach((c, idx) \=\> {  
        lines \+= \`set policy-options policy-statement POL-UPSTREAM-OUT term TERM-PERMIT then community add COMM-POLICY-\${idx \+ 1}\\n\`;  
      });  
      largeComms.forEach((lc, idx) \=\> {  
        lines \+= \`set policy-options policy-statement POL-UPSTREAM-OUT term TERM-PERMIT then community add COMM-LARGE-\${idx \+ 1}\\n\`;  
      });

      if (prepends \> 0\) {  
        const prepStr \= Array(prepends).fill(asn).join(' ');  
        lines \+= \`set policy-options policy-statement POL-UPSTREAM-OUT term TERM-PERMIT then as-path-prepend "\${prepStr}"\\n\`;  
      }

      lines \+= \`set policy-options policy-statement POL-UPSTREAM-OUT term TERM-PERMIT then accept\\n\`;  
      lines \+= \`set policy-options policy-statement POL-UPSTREAM-OUT term TERM-LEAK-SHIELD then reject\\n\\n\`;

      lines \+= \`set protocols bgp group UPSTREAM-TRANSIT peer-as \${peerAsn}\\n\`;  
      lines \+= \`set protocols bgp group UPSTREAM-TRANSIT neighbor \${peerIp} export POL-UPSTREAM-OUT\\n\`;  
      if (p.stripPrivateAsn \!== false) {  
        lines \+= \`set protocols bgp group UPSTREAM-TRANSIT neighbor \${peerIp} remove-private\\n\`;  
      }

      return lines;  
    },

    generateArista(p) {  
      const asn \= p.localAsn || 65001;  
      const peerAsn \= p.targetAsn || 3356;  
      const peerIp \= p.peerIp || '192.0.2.1';  
      const prefixes \= p.prefixes || \['198.51.100.0/24'\];  
      const prepends \= parseInt(p.asPathPrepend, 10\) || 0;  
      const comms \= (p.communities || \[\]).map(c \=\> c.toLowerCase() \=== 'no\_export' ? 'no-export' : c);  
      const largeComms \= p.largeCommunities || \[\];

      let prefixLines \= '';  
      prefixes.forEach((pref, idx) \=\> {  
        prefixLines \+= \`ip prefix-list PL-BGP-OUT seq \${(idx \+ 1\) \* 10} permit \${pref}\\n\`;  
      });

      let prependClause \= '';  
      if (prepends \> 0\) {  
        const prepStr \= Array(prepends).fill(asn).join(' ');  
        prependClause \= \`   set as-path prepend \${prepStr}\\n\`;  
      }

      let commClause \= '';  
      if (comms.length \> 0\) {  
        commClause \+= \`   set community \${comms.join(' ')} additive\\n\`;  
      }  
      if (largeComms.length \> 0\) {  
        commClause \+= \`   set large-community \${largeComms.join(' ')} additive\\n\`;  
      }

      return \`\! \========================================================  
\! Arista EOS BGP Policy & Route-Map  
\! \========================================================  
\${prefixLines}\!  
route-map RM-UPSTREAM-OUT permit 10  
   match ip address prefix-list PL-BGP-OUT  
\${commClause}\${prependClause}\!  
route-map RM-UPSTREAM-OUT deny 99  
\!  
router bgp \${asn}  
   neighbor \${peerIp} remote-as \${peerAsn}  
   neighbor \${peerIp} route-map RM-UPSTREAM-OUT out  
   neighbor \${peerIp} send-community standard large  
\${p.stripPrivateAsn \!== false ? \`   neighbor \${peerIp} remove-private-as\\n\` : ''}\`;  
    },

    generateBird(p) {  
      const asn \= p.localAsn || 65001;  
      const peerAsn \= p.targetAsn || 3356;  
      const peerIp \= p.peerIp || '192.0.2.1';  
      const prefixes \= p.prefixes || \['198.51.100.0/24'\];  
      const prepends \= parseInt(p.asPathPrepend, 10\) || 0;  
      const comms \= p.communities || \[\];  
      const largeComms \= p.largeCommunities || \[\];

      let commAdditions \= '';  
      comms.forEach(c \=\> {  
        if (c.toLowerCase() \=== 'no\_export') {  
          commAdditions \+= \`        bgp\_community.add((65535, 65281)); \# NO\_EXPORT\\n\`;  
        } else if (c.includes(':')) {  
          const parts \= c.split(':');  
          commAdditions \+= \`        bgp\_community.add((\${parts\[0\]}, \${parts\[1\]}));\\n\`;  
        }  
      });

      largeComms.forEach(lc \=\> {  
        const parts \= lc.split(':');  
        commAdditions \+= \`        bgp\_large\_community.add((\${parts\[0\]}, \${parts\[1\]}, \${parts\[2\]}));\\n\`;  
      });

      let prependClause \= '';  
      if (prepends \> 0\) {  
        for (let i \= 0; i \< prepends; i++) {  
          prependClause \+= \`        bgp\_path.prepend(\${asn});\\n\`;  
        }  
      }

      const prefixListStr \= prefixes.join(', ');

      return \`\# \========================================================  
\# BIRD 2.x Routing Daemon Policy  
\# \========================================================

filter filter\_upstream\_out {  
    if (net \~ \[ \${prefixListStr} \]) then {  
\${commAdditions}\${prependClause}        accept;  
    }  
    reject; \# Prevent Route Leaks  
}

protocol bgp upstream\_transit\_\${peerAsn} {  
    local as \${asn};  
    neighbor \${peerIp} as \${peerAsn};  
    ipv4 {  
        import none;  
        export filter filter\_upstream\_out;  
    };  
}\`;  
    },

    generateFRR(p) {  
      const asn \= p.localAsn || 65001;  
      const peerAsn \= p.targetAsn || 3356;  
      const peerIp \= p.peerIp || '192.0.2.1';  
      const prefixes \= p.prefixes || \['198.51.100.0/24'\];  
      const prepends \= parseInt(p.asPathPrepend, 10\) || 0;  
      const comms \= (p.communities || \[\]).map(c \=\> c.toLowerCase() \=== 'no\_export' ? 'no-export' : c);

      let prefixLines \= '';  
      prefixes.forEach((pref, idx) \=\> {  
        prefixLines \+= \`ip prefix-list PL-BGP-OUT seq \${(idx \+ 1\) \* 10} permit \${pref}\\n\`;  
      });

      let prependClause \= '';  
      if (prepends \> 0\) {  
        const prepStr \= Array(prepends).fill(asn).join(' ');  
        prependClause \= \` set as-path prepend \${prepStr}\\n\`;  
      }

      let commClause \= '';  
      if (comms.length \> 0\) {  
        commClause \+= \` set community \${comms.join(' ')} additive\\n\`;  
      }

      return \`\! \========================================================  
\! FRRouting (FRR) BGP Policy  
\! \========================================================  
\${prefixLines}\!  
route-map RM-UPSTREAM-OUT permit 10  
 match ip address prefix-list PL-BGP-OUT  
\${commClause}\${prependClause}\!  
route-map RM-UPSTREAM-OUT deny 99  
\!  
router bgp \${asn}  
 neighbor \${peerIp} remote-as \${peerAsn}  
 neighbor \${peerIp} route-map RM-UPSTREAM-OUT out  
 neighbor \${peerIp} send-community both\`;  
    }  
  };

  return BGPVendorSynthesizer;  
});  
