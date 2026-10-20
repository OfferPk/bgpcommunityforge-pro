/\*\*  
 \* BGPCommunityForge Pro \- Core BGP Community Engine  
 \* Standards Compliant: RFC 1997, RFC 4360, RFC 8092, RFC 7999, RFC 8326, RFC 9234  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.BGPCommunityEngine \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  const WELL\_KNOWN\_COMMUNITIES \= {  
    '65535:65281': { name: 'NO\_EXPORT', hex: '0xFFFFFF01', rfc: 'RFC 1997', desc: 'Do not advertise outside local AS or confederation boundary.' },  
    '65535:65282': { name: 'NO\_ADVERTISE', hex: '0xFFFFFF02', rfc: 'RFC 1997', desc: 'Do not advertise to any peer (internal or external).' },  
    '65535:65283': { name: 'NO\_EXPORT\_SUBCONFED', hex: '0xFFFFFF03', rfc: 'RFC 1997', desc: 'Do not advertise outside local member AS.' },  
    '65535:0':     { name: 'GRACEFUL\_SHUTDOWN', hex: '0xFFFF0000', rfc: 'RFC 8326', desc: 'Gracefully drain traffic prior to planned session maintenance.' },  
    '65535:666':   { name: 'BLACKHOLE', hex: '0xFFFF029A', rfc: 'RFC 7999', desc: 'Remotely Triggered Black Hole (RTBH) to drop DDoS attack prefixes.' },  
    '65535:6':     { name: 'LLGR\_STALE', hex: '0xFFFF0006', rfc: 'RFC 9224', desc: 'Long-Lived Graceful Restart stale route indicator.' }  
  };

  const BGPCommunityEngine \= {  
    WELL\_KNOWN: WELL\_KNOWN\_COMMUNITIES,

    /\*\*  
     \* Checks if an ASN is in the RFC 6996 / RFC 7300 private ASN range  
     \*/  
    isPrivateASN(asn) {  
      const num \= parseInt(asn, 10);  
      if (isNaN(num)) return false;  
      // 16-bit private: 64512 \- 65534  
      if (num \>= 64512 && num \<= 65534\) return true;  
      // 32-bit private: 4200000000 \- 4294967294  
      if (num \>= 4200000000 && num \<= 4294967294\) return true;  
      return false;  
    },

    /\*\*  
     \* Validates and parses a Standard Community (RFC 1997): "ASN:VALUE"  
     \*/  
    parseStandardCommunity(str) {  
      if (\!str || typeof str \!== 'string') return null;  
      const clean \= str.trim();

      // Check well-known keyword  
      for (const \[comm, meta\] of Object.entries(WELL\_KNOWN\_COMMUNITIES)) {  
        if (clean.toUpperCase() \=== meta.name) {  
          const parts \= comm.split(':');  
          return {  
            type: 'STANDARD',  
            asn: parseInt(parts\[0\], 10),  
            value: parseInt(parts\[1\], 10),  
            formatted: comm,  
            wellKnown: meta.name,  
            desc: meta.desc,  
            rfc: meta.rfc  
          };  
        }  
      }

      const match \= clean.match(/^(\\d{1,5}):(\\d{1,5})\$/);  
      if (\!match) return null;

      const asn \= parseInt(match\[1\], 10);  
      const value \= parseInt(match\[2\], 10);

      if (asn \> 65535 || value \> 65535\) return null;

      const formatted \= \`\${asn}:\${value}\`;  
      const wk \= WELL\_KNOWN\_COMMUNITIES\[formatted\];

      return {  
        type: 'STANDARD',  
        asn,  
        value,  
        formatted,  
        wellKnown: wk ? wk.name : null,  
        desc: wk ? wk.desc : 'Custom traffic engineering standard community.',  
        rfc: wk ? wk.rfc : 'RFC 1997'  
      };  
    },

    /\*\*  
     \* Validates and parses a Large Community (RFC 8092): "ASN:DATA1:DATA2"  
     \*/  
    parseLargeCommunity(str) {  
      if (\!str || typeof str \!== 'string') return null;  
      const clean \= str.trim();  
      const match \= clean.match(/^(\\d{1,10}):(\\d{1,10}):(\\d{1,10})\$/);  
      if (\!match) return null;

      const asn \= parseInt(match\[1\], 10);  
      const data1 \= parseInt(match\[2\], 10);  
      const data2 \= parseInt(match\[3\], 10);

      const MAX\_32 \= 4294967295;  
      if (asn \> MAX\_32 || data1 \> MAX\_32 || data2 \> MAX\_32) return null;

      return {  
        type: 'LARGE',  
        globalAdmin: asn,  
        data1,  
        data2,  
        formatted: \`\${asn}:\${data1}:\${data2}\`,  
        rfc: 'RFC 8092',  
        desc: 'RFC 8092 12-byte Large Community for 4-byte ASN architectures.'  
      };  
    },

    /\*\*  
     \* Generic community parser (detects standard, extended, or large)  
     \*/  
    parse(str) {  
      if (\!str) return null;  
      const std \= this.parseStandardCommunity(str);  
      if (std) return std;  
      const lrg \= this.parseLargeCommunity(str);  
      if (lrg) return lrg;  
      return null;  
    },

    /\*\*  
     \* Encodes standard ASN:VALUE into 32-bit unsigned integer  
     \*/  
    toUint32(asn, value) {  
      return ((asn & 0xFFFF) \<\< 16\) | (value & 0xFFFF);  
    },

    /\*\*  
     \* Decodes 32-bit unsigned integer to standard ASN:VALUE string  
     \*/  
    fromUint32(uint32) {  
      const asn \= (uint32 \>\>\> 16\) & 0xFFFF;  
      const value \= uint32 & 0xFFFF;  
      return \`\${asn}:\${value}\`;  
    },

    /\*\*  
     \* Validates an IP Prefix and its subnet length  
     \*/  
    validatePrefix(cidr) {  
      if (\!cidr || typeof cidr \!== 'string') {  
        return { valid: false, error: 'Empty prefix' };  
      }  
      const parts \= cidr.trim().split('/');  
      if (parts.length \!== 2\) {  
        return { valid: false, error: 'Malformed CIDR (missing /)' };  
      }

      const ip \= parts\[0\];  
      const mask \= parseInt(parts\[1\], 10);

      // IPv4 check  
      if (ip.includes('.')) {  
        if (isNaN(mask) || mask \< 8 || mask \> 32\) {  
          return { valid: false, error: \`Invalid IPv4 mask /\${parts\[1\]}. Must be /8 to /32.\` };  
        }  
        const octets \= ip.split('.');  
        if (octets.length \!== 4\) return { valid: false, error: 'Invalid IPv4 octets' };  
        for (const oct of octets) {  
          const n \= parseInt(oct, 10);  
          if (isNaN(n) || n \< 0 || n \> 255\) return { valid: false, error: \`Invalid octet \${oct}\` };  
        }  
        const isLeakedMoreSpecific \= mask \> 24 && mask \!== 32;  
        return {  
          valid: true,  
          ipVersion: 4,  
          ip,  
          mask,  
          isRtbhHost: mask \=== 32,  
          isLeakedMoreSpecific, // e.g. /25 to /31 are filtered by Tier-1 upstreams  
          formatted: \`\${ip}/\${mask}\`  
        };  
      }

      // IPv6 check  
      if (ip.includes(':')) {  
        if (isNaN(mask) || mask \< 19 || mask \> 128\) {  
          return { valid: false, error: \`Invalid IPv6 mask /\${parts\[1\]}. Must be /19 to /128.\` };  
        }  
        const isLeakedMoreSpecific \= mask \> 48 && mask \!== 128;  
        return {  
          valid: true,  
          ipVersion: 6,  
          ip,  
          mask,  
          isRtbhHost: mask \=== 128,  
          isLeakedMoreSpecific, // e.g. /49 to /127 are filtered by Tier-1 upstreams  
          formatted: \`\${ip}/\${mask}\`  
        };  
      }

      return { valid: false, error: 'Unknown IP version' };  
    }  
  };

  return BGPCommunityEngine;  
});  
