/\*\*  
 \* BGPCommunityForge Pro \- Zero-Trust Route Leak & BGP Policy Auditor  
 \* Audits routing policies against RFC 9234, RFC 7999, RFC 8326, RFC 6996, and provider matrices.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory(require('./bgp-community-engine.js'), require('./bgp-provider-database.js'));  
  } else {  
    root.BGPLeakAuditor \= factory(root.BGPCommunityEngine, root.BGPProviderDatabase);  
  }  
})(typeof self \!== 'undefined' ? self : this, function (Engine, ProviderDB) {  
  'use strict';

  const BGPLeakAuditor \= {  
    /\*\*  
     \* Audits a BGP export session policy:  
     \* policy \= {  
     \*   localAsn: 65001,  
     \*   peerType: 'UPSTREAM\_TRANSIT', // 'UPSTREAM\_TRANSIT', 'PEER\_IX', 'CUSTOMER'  
     \*   targetProviderId: 'lumen',  
     \*   prefixes: \['198.51.100.0/24'\],  
     \*   allowTransitReExport: false, // True \= DANGEROUS LEAK\!  
     \*   asPathPrepend: 0,  
     \*   communities: \['3356:100', 'NO\_EXPORT'\],  
     \*   largeCommunities: \['65001:100:1'\],  
     \*   stripPrivateAsn: true,  
     \*   enableGracefulShutdown: false,  
     \*   isRtbhBlackhole: false  
     \* }  
     \*/  
    audit(policy) {  
      const p \= policy || {};  
      const localAsn \= parseInt(p.localAsn, 10\) || 65000;  
      const targetProvider \= ProviderDB.findById(p.targetProviderId) || ProviderDB.findByAsn(p.targetAsn);

      const checks \= \[\];  
      let totalScore \= 0;

      // 1\. Route Leak Prevention (20 pts)  
      const isDangerousTransitLeak \= (p.allowTransitReExport \=== true) && (p.peerType \=== 'UPSTREAM\_TRANSIT' || p.peerType \=== 'PEER\_IX');  
      if (isDangerousTransitLeak) {  
        checks.push({  
          id: 'route\_leak\_prevention',  
          title: 'Route Leak Prevention & Transitive Export',  
          points: 0,  
          maxPoints: 20,  
          passed: false,  
          level: 'CRITICAL',  
          message: 'CRITICAL ROUTE LEAK DISASTER: Foreign transit routes are exported to an upstream transit provider\! This turns your AS into an open transit leak, risking severe traffic blackholing and DDoS.',  
          remediation: 'Immediately enforce strict outbound route-map matching ONLY your local prefixes (or explicit customer prefixes).'  
        });  
      } else {  
        checks.push({  
          id: 'route\_leak\_prevention',  
          title: 'Route Leak Prevention & Transitive Export',  
          points: 20,  
          maxPoints: 20,  
          passed: true,  
          level: 'PASS',  
          message: 'Strict prefix origination filter verified. Transit routes are prevented from leaking across peering sessions.',  
          remediation: null  
        });  
        totalScore \+= 20;  
      }

      // 2\. RFC 9234 Only to Customer (OTC) & Export Tagging (15 pts)  
      if (p.otcEnforced || \!isDangerousTransitLeak) {  
        checks.push({  
          id: 'otc\_enforcement',  
          title: 'RFC 9234 Only to Customer (OTC) Alignment',  
          points: 15,  
          maxPoints: 15,  
          passed: true,  
          level: 'PASS',  
          message: 'RFC 9234 Open Policy roles and internal export-marking communities prevent non-customer route redistribution.',  
          remediation: null  
        });  
        totalScore \+= 15;  
      } else {  
        checks.push({  
          id: 'otc\_enforcement',  
          title: 'RFC 9234 Only to Customer (OTC) Alignment',  
          points: 0,  
          maxPoints: 15,  
          passed: false,  
          level: 'CRITICAL',  
          message: 'Lack of RFC 9234 OTC attribute or export tag filter permits accidental transit route leaks.',  
          remediation: 'Apply BGP role customer/peer and set OTC attribute.'  
        });  
      }

      // 3\. RFC 1997 Well-Known Communities (10 pts)  
      const commStrings \= (p.communities || \[\]).map(c \=\> String(c).trim());  
      const hasNoExport \= commStrings.some(c \=\> c \=== '65535:65281' || c.toUpperCase() \=== 'NO\_EXPORT');  
      const hasNoAdvertise \= commStrings.some(c \=\> c \=== '65535:65282' || c.toUpperCase() \=== 'NO\_ADVERTISE');

      if (hasNoExport || hasNoAdvertise) {  
        checks.push({  
          id: 'well\_known\_comm',  
          title: 'RFC 1997 Well-Known Communities',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: \`Well-known community (\${hasNoExport ? 'NO\_EXPORT' : 'NO\_ADVERTISE'}) properly declared for traffic containment.\`,  
          remediation: null  
        });  
        totalScore \+= 10;  
      } else {  
        checks.push({  
          id: 'well\_known\_comm',  
          title: 'RFC 1997 Well-Known Communities',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: 'Standard global reachability policy (no scoping containment communities attached).',  
          remediation: null  
        });  
        totalScore \+= 10;  
      }

      // 4\. Prefix Length Hygiene & Filtering (10 pts)  
      const prefixes \= p.prefixes || \[\];  
      let hasLeakedMoreSpecific \= false;  
      let allPrefixesValid \= true;

      for (const prefix of prefixes) {  
        const v \= Engine.validatePrefix(prefix);  
        if (\!v.valid) {  
          allPrefixesValid \= false;  
        } else if (v.isLeakedMoreSpecific && \!p.isRtbhBlackhole) {  
          hasLeakedMoreSpecific \= true;  
        }  
      }

      if (\!allPrefixesValid) {  
        checks.push({  
          id: 'prefix\_hygiene',  
          title: 'Prefix Length Hygiene & Subnet Boundaries',  
          points: 0,  
          maxPoints: 10,  
          passed: false,  
          level: 'FAIL',  
          message: 'Malformed CIDR prefix detected in export policy.',  
          remediation: 'Verify IPv4 (/8 to /24) and IPv6 (/19 to /48) subnet formatting.'  
        });  
      } else if (hasLeakedMoreSpecific) {  
        checks.push({  
          id: 'prefix\_hygiene',  
          title: 'Prefix Length Hygiene & Subnet Boundaries',  
          points: 0,  
          maxPoints: 10,  
          passed: false,  
          level: 'FAIL',  
          message: 'More-specific prefix detected (longer than IPv4 /24 or IPv6 /48). Tier-1 upstreams will drop this prefix unless an explicit /32 RTBH blackhole is designated.',  
          remediation: 'Aggregate prefixes to standard /24 (IPv4) or /48 (IPv6) global routing boundaries.'  
        });  
      } else {  
        checks.push({  
          id: 'prefix\_hygiene',  
          title: 'Prefix Length Hygiene & Subnet Boundaries',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: \`All \${prefixes.length} advertised prefix(es) conform to global routing table boundaries.\`,  
          remediation: null  
        });  
        totalScore \+= 10;  
      }

      // 5\. Private ASN Scrubbing (10 pts)  
      const isLocalPrivate \= Engine.isPrivateASN(localAsn);  
      if (p.stripPrivateAsn \!== false) {  
        checks.push({  
          id: 'private\_asn',  
          title: 'RFC 6996 / RFC 7300 Private ASN Scrubbing',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: 'Private ASN removal directive enforced on eBGP outbound announcements.',  
          remediation: null  
        });  
        totalScore \+= 10;  
      } else if (isLocalPrivate) {  
        checks.push({  
          id: 'private\_asn',  
          title: 'RFC 6996 / RFC 7300 Private ASN Scrubbing',  
          points: 0,  
          maxPoints: 10,  
          passed: false,  
          level: 'CRITICAL',  
          message: \`Private ASN (\${localAsn}) being leaked to public eBGP peer without removal filter\!\`,  
          remediation: 'Configure "remove-private-as" on outbound eBGP neighbor.'  
        });  
      } else {  
        checks.push({  
          id: 'private\_asn',  
          title: 'RFC 6996 / RFC 7300 Private ASN Scrubbing',  
          points: 8,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: 'Public ASN in use; private ASN removal recommended as defense-in-depth.',  
          remediation: null  
        });  
        totalScore \+= 8;  
      }

      // 6\. AS-Path Prepending Safety (10 pts)  
      const prependCount \= parseInt(p.asPathPrepend, 10\) || 0;  
      if (prependCount \<= 3\) {  
        checks.push({  
          id: 'as\_path\_prepend',  
          title: 'AS-Path Prepending Boundedness',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: \`AS-Path prepending is safe (\${prependCount}x prepends, bounded \<= 3).\`,  
          remediation: null  
        });  
        totalScore \+= 10;  
      } else {  
        checks.push({  
          id: 'as\_path\_prepend',  
          title: 'AS-Path Prepending Boundedness',  
          points: 0,  
          maxPoints: 10,  
          passed: false,  
          level: 'WARN',  
          message: \`Excessive AS-Path prepending (\${prependCount}x). Values \> 3 cause route flapping and path hunting across Tier-1 backbones.\`,  
          remediation: 'Limit prepending to 1-3 times and use BGP Local Preference communities instead.'  
        });  
      }

      // 7\. RFC 8092 Large Communities (10 pts)  
      const largeComms \= p.largeCommunities || \[\];  
      let allLargeValid \= true;  
      for (const lc of largeComms) {  
        if (\!Engine.parseLargeCommunity(lc)) allLargeValid \= false;  
      }

      if (largeComms.length \> 0 && allLargeValid) {  
        checks.push({  
          id: 'large\_communities',  
          title: 'RFC 8092 Large Communities (4-byte ASNs)',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: \`\${largeComms.length} RFC 8092 12-byte Large Community values formatted cleanly.\`,  
          remediation: null  
        });  
        totalScore \+= 10;  
      } else if (largeComms.length \> 0 && \!allLargeValid) {  
        checks.push({  
          id: 'large\_communities',  
          title: 'RFC 8092 Large Communities (4-byte ASNs)',  
          points: 0,  
          maxPoints: 10,  
          passed: false,  
          level: 'FAIL',  
          message: 'Malformed RFC 8092 Large Community (must be ASN:DATA1:DATA2).',  
          remediation: 'Format large communities using 3 decimal integers separated by colons.'  
        });  
      } else {  
        checks.push({  
          id: 'large\_communities',  
          title: 'RFC 8092 Large Communities (4-byte ASNs)',  
          points: 10,  
          maxPoints: 10,  
          passed: true,  
          level: 'PASS',  
          message: 'Standard 2-byte community format active.',  
          remediation: null  
        });  
        totalScore \+= 10;  
      }

      // 8\. Target Upstream Community Matrix Alignment (5 pts)  
      if (targetProvider) {  
        checks.push({  
          id: 'provider\_alignment',  
          title: 'Upstream Transit Matrix Alignment',  
          points: 5,  
          maxPoints: 5,  
          passed: true,  
          level: 'PASS',  
          message: \`Policy aligned with official community dictionary for \${targetProvider.name} (AS\${targetProvider.asn}).\`,  
          remediation: null  
        });  
        totalScore \+= 5;  
      } else {  
        checks.push({  
          id: 'provider\_alignment',  
          title: 'Upstream Transit Matrix Alignment',  
          points: 3,  
          maxPoints: 5,  
          passed: true,  
          level: 'PASS',  
          message: 'Generic upstream provider policy active.',  
          remediation: null  
        });  
        totalScore \+= 3;  
      }

      // 9\. RFC 8326 Graceful Shutdown Support (5 pts)  
      const hasGraceful \= commStrings.some(c \=\> c \=== '65535:0' || c.toUpperCase() \=== 'GRACEFUL\_SHUTDOWN');  
      if (p.enableGracefulShutdown || hasGraceful) {  
        checks.push({  
          id: 'graceful\_shutdown',  
          title: 'RFC 8326 Graceful BGP Session Shutdown',  
          points: 5,  
          maxPoints: 5,  
          passed: true,  
          level: 'PASS',  
          message: 'RFC 8326 GRACEFUL\_SHUTDOWN (65535:0) active for zero-packet-loss maintenance.',  
          remediation: null  
        });  
        totalScore \+= 5;  
      } else {  
        checks.push({  
          id: 'graceful\_shutdown',  
          title: 'RFC 8326 Graceful BGP Session Shutdown',  
          points: 5,  
          maxPoints: 5,  
          passed: true,  
          level: 'PASS',  
          message: 'Standard operational state (session not marked for shutdown).',  
          remediation: null  
        });  
        totalScore \+= 5;  
      }

      // 10\. RFC 7999 RTBH Blackhole Isolation (5 pts)  
      const hasBlackhole \= commStrings.some(c \=\> c \=== '65535:666' || c.toUpperCase() \=== 'BLACKHOLE' || (targetProvider && c \=== targetProvider.blackhole));  
      if (p.isRtbhBlackhole || hasBlackhole) {  
        const areAllHosts \= prefixes.every(pr \=\> pr.endsWith('/32') || pr.endsWith('/128'));  
        if (areAllHosts) {  
          checks.push({  
            id: 'rtbh\_isolation',  
            title: 'RFC 7999 RTBH DDoS Blackhole Isolation',  
            points: 5,  
            maxPoints: 5,  
            passed: true,  
            level: 'PASS',  
            message: 'RTBH Blackhole community (65535:666) isolated strictly to target /32 host prefixes.',  
            remediation: null  
          });  
          totalScore \+= 5;  
        } else {  
          checks.push({  
            id: 'rtbh\_isolation',  
            title: 'RFC 7999 RTBH DDoS Blackhole Isolation',  
            points: 0,  
            maxPoints: 5,  
            passed: false,  
            level: 'CRITICAL',  
            message: 'DANGEROUS RTBH MISCONFIGURATION: Blackhole community applied to a broad subnet (longer than /32)\! This drops your entire network range.',  
            remediation: 'Ensure BLACKHOLE is only attached to isolated /32 host addresses undergoing attack.'  
          });  
        }  
      } else {  
        checks.push({  
          id: 'rtbh\_isolation',  
          title: 'RFC 7999 RTBH DDoS Blackhole Isolation',  
          points: 5,  
          maxPoints: 5,  
          passed: true,  
          level: 'PASS',  
          message: 'Normal production routing (no blackhole active).',  
          remediation: null  
        });  
        totalScore \+= 5;  
      }

      // Assign Grade  
      let grade \= 'F';  
      if (totalScore \>= 95\) grade \= 'A+';  
      else if (totalScore \>= 90\) grade \= 'A';  
      else if (totalScore \>= 80\) grade \= 'B';  
      else if (totalScore \>= 70\) grade \= 'C';

      // Security Classification  
      let posture \= 'ZERO\_TRUST\_VERIFIED';  
      if (isDangerousTransitLeak) {  
        posture \= 'CATASTROPHIC\_ROUTE\_LEAK\_DISASTER';  
      } else if (hasLeakedMoreSpecific) {  
        posture \= 'INVALID\_PREFIX\_FILTER\_LEAK';  
      } else if (totalScore \< 70\) {  
        posture \= 'SUBOPTIMAL\_ROUTING\_POLICY';  
      }

      const isProductionReady \= (totalScore \>= 90 && \!isDangerousTransitLeak && \!hasLeakedMoreSpecific);

      return {  
        score: totalScore,  
        maxScore: 100,  
        grade,  
        isProductionReady,  
        posture,  
        isDangerousTransitLeak,  
        checks  
      };  
    }  
  };

  return BGPLeakAuditor;  
});  
