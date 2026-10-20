/\*\*  
 \* BGPCommunityForge Pro \- Tier-1 Upstream Provider Community Database  
 \* Curated mappings for Lumen (Level3), Cogent, Arelion (Telia), NTT, HE, and Tata.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.BGPProviderDatabase \= factory();  
  }  
})(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  const PROVIDERS \= \[  
    {  
      id: 'lumen',  
      asn: 3356,  
      name: 'Lumen (CenturyLink / Level 3)',  
      tier: 'Tier-1',  
      description: 'Global Tier-1 backbone network with granular regional & local preference control.',  
      localPref: \[  
        { community: '3356:70', value: 70, label: 'Low Preference (Backup)' },  
        { community: '3356:80', value: 80, label: 'Secondary Route' },  
        { community: '3356:90', value: 90, label: 'Tertiary Preference' },  
        { community: '3356:100', value: 100, label: 'Default Customer Route' },  
        { community: '3356:110', value: 110, label: 'High Preference (Primary)' }  
      \],  
      prepending: \[  
        { community: '3356:2011', times: 1, label: 'Prepend 1x to all peers' },  
        { community: '3356:2012', times: 2, label: 'Prepend 2x to all peers' },  
        { community: '3356:2013', times: 3, label: 'Prepend 3x to all peers' }  
      \],  
      scopeControl: \[  
        { community: '3356:2000', label: 'Do not announce to peers outside Lumen' },  
        { community: '3356:2050', label: 'Do not announce to peers in Europe' },  
        { community: '3356:2060', label: 'Do not announce to peers in Asia' }  
      \],  
      blackhole: '3356:9999'  
    },  
    {  
      id: 'cogent',  
      asn: 174,  
      name: 'Cogent Communications',  
      tier: 'Tier-1',  
      description: 'High-volume multinational Tier-1 carrier with simple numerical community policy.',  
      localPref: \[  
        { community: '174:10', value: 10, label: 'Lowest / Emergency Backup' },  
        { community: '174:70', value: 70, label: 'Backup Link' },  
        { community: '174:100', value: 100, label: 'Default Customer Route' },  
        { community: '174:120', value: 120, label: 'Primary Link' },  
        { community: '174:130', value: 130, label: 'Highest Preference' }  
      \],  
      prepending: \[  
        { community: '174:3001', times: 1, label: 'Prepend 1x to all Cogent peers' },  
        { community: '174:3002', times: 2, label: 'Prepend 2x to all Cogent peers' },  
        { community: '174:3003', times: 3, label: 'Prepend 3x to all Cogent peers' }  
      \],  
      scopeControl: \[  
        { community: '174:2000', label: 'Do not advertise to Cogent peers' }  
      \],  
      blackhole: '174:666'  
    },  
    {  
      id: 'arelion',  
      asn: 1299,  
      name: 'Arelion (formerly Telia Carrier)',  
      tier: 'Tier-1',  
      description: 'Top-ranked global Internet backbone operator with rich regional communities.',  
      localPref: \[  
        { community: '1299:70', value: 70, label: 'Low Preference' },  
        { community: '1299:80', value: 80, label: 'Secondary Route' },  
        { community: '1299:90', value: 90, label: 'Backup Route' },  
        { community: '1299:100', value: 100, label: 'Default Preference' }  
      \],  
      prepending: \[  
        { community: '1299:1', times: 1, label: 'Prepend 1x globally' },  
        { community: '1299:2,1299:2', times: 2, label: 'Prepend 2x globally' },  
        { community: '1299:3,1299:3,1299:3', times: 3, label: 'Prepend 3x globally' }  
      \],  
      scopeControl: \[  
        { community: '1299:3000', label: 'Do not announce in North America' },  
        { community: '1299:3001', label: 'Do not announce in Europe' }  
      \],  
      blackhole: '1299:999'  
    },  
    {  
      id: 'ntt',  
      asn: 2914,  
      name: 'NTT Communications (GIN)',  
      tier: 'Tier-1',  
      description: 'Tier-1 Global IP Network (GIN) connecting Asia, Americas, and Europe.',  
      localPref: \[  
        { community: '2914:410', value: 70, label: 'Low Preference (LP 70)' },  
        { community: '2914:420', value: 80, label: 'Secondary (LP 80)' },  
        { community: '2914:430', value: 90, label: 'Tertiary (LP 90)' },  
        { community: '2914:440', value: 100, label: 'Default Customer Route (LP 100)' }  
      \],  
      prepending: \[  
        { community: '2914:411', times: 1, label: 'Prepend 1x to NTT peers' },  
        { community: '2914:412', times: 2, label: 'Prepend 2x to NTT peers' },  
        { community: '2914:413', times: 3, label: 'Prepend 3x to NTT peers' }  
      \],  
      scopeControl: \[  
        { community: '2914:429', label: 'Do not announce to NTT peers' }  
      \],  
      blackhole: '2914:666'  
    },  
    {  
      id: 'he',  
      asn: 6939,  
      name: 'Hurricane Electric',  
      tier: 'Tier-1/Transit',  
      description: 'World largest IPv6 transit backbone with extensive public peering presence.',  
      localPref: \[  
        { community: '6939:70', value: 70, label: 'Local Preference 70' },  
        { community: '6939:80', value: 80, label: 'Local Preference 80' },  
        { community: '6939:90', value: 90, label: 'Local Preference 90' },  
        { community: '6939:100', value: 100, label: 'Default Customer Route' }  
      \],  
      prepending: \[  
        { community: '6939:65', times: 1, label: 'Prepend 1x to peers' },  
        { community: '6939:66', times: 2, label: 'Prepend 2x to peers' },  
        { community: '6939:67', times: 3, label: 'Prepend 3x to peers' }  
      \],  
      scopeControl: \[\],  
      blackhole: '6939:666'  
    },  
    {  
      id: 'tata',  
      asn: 6453,  
      name: 'Tata Communications',  
      tier: 'Tier-1',  
      description: 'Major global Tier-1 transit provider with substantial trans-subsea connectivity.',  
      localPref: \[  
        { community: '6453:70', value: 70, label: 'Local Preference 70' },  
        { community: '6453:80', value: 80, label: 'Local Preference 80' },  
        { community: '6453:90', value: 90, label: 'Local Preference 90' },  
        { community: '6453:100', value: 100, label: 'Default Customer Route' }  
      \],  
      prepending: \[  
        { community: '6453:101', times: 1, label: 'Prepend 1x to all peers' },  
        { community: '6453:102', times: 2, label: 'Prepend 2x to all peers' },  
        { community: '6453:103', times: 3, label: 'Prepend 3x to all peers' }  
      \],  
      scopeControl: \[\],  
      blackhole: '6453:666'  
    }  
  \];

  const BGPProviderDatabase \= {  
    PROVIDERS,

    findById(id) {  
      return PROVIDERS.find(p \=\> p.id \=== id) || null;  
    },

    findByAsn(asn) {  
      const num \= parseInt(asn, 10);  
      return PROVIDERS.find(p \=\> p.asn \=== num) || null;  
    },

    lookupCommunity(communityStr) {  
      if (\!communityStr) return null;  
      const clean \= communityStr.trim();  
      const parts \= clean.split(':');  
      if (parts.length \< 2\) return null;

      const targetAsn \= parseInt(parts\[0\], 10);  
      const provider \= this.findByAsn(targetAsn);  
      if (\!provider) return null;

      // Check LP  
      const lp \= (provider.localPref || \[\]).find(x \=\> x.community \=== clean);  
      if (lp) return { provider, category: 'LOCAL\_PREF', detail: lp };

      // Check prepending  
      const prep \= (provider.prepending || \[\]).find(x \=\> x.community.includes(clean));  
      if (prep) return { provider, category: 'PREPEND', detail: prep };

      // Check scope  
      const sc \= (provider.scopeControl || \[\]).find(x \=\> x.community \=== clean);  
      if (sc) return { provider, category: 'SCOPE\_CONTROL', detail: sc };

      // Check blackhole  
      if (provider.blackhole \=== clean) {  
        return { provider, category: 'BLACKHOLE', detail: { label: 'Provider RTBH Blackhole' } };  
      }

      return { provider, category: 'CUSTOM', detail: null };  
    }  
  };

  return BGPProviderDatabase;  
});  
