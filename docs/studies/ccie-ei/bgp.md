# BGP

## The BGP table
```
R1#show bgp ipv4 uni  
BGP table version is 18, local router ID is 100.1.3.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
     0.0.0.0          0.0.0.0                                0 i
 *>  1.1.1.1/32       0.0.0.0                  0         32768 i
 *b  2.2.2.2/32       13.1.1.3                               0 300 200 i
 *>                   12.1.1.2                 0             0 200 i
 r>i 4.0.0.0          14.1.1.4                 0    100      0 i
 *b  21.0.0.0         13.1.1.3                               0 300 200 ?
 *>                   12.1.1.2                 0             0 200 ?
 *b  22.0.0.0         13.1.1.3                               0 300 200 i
 *>                   12.1.1.2                 0             0 200 i
 *m  23.1.1.0/24      12.1.1.2                 0             0 200 i
 *>                   13.1.1.3                 0             0 300 i
 s>  100.1.0.0/24     0.0.0.0                  0         32768 i
 *>  100.1.0.0/22     0.0.0.0                            32768 i
 s>  100.1.1.0/24     0.0.0.0                  0         32768 i
```

#### Column 1
 - `*` valid
 - `r` RIB-failure
 - `m` multipathing
 - `s` supressed: suppressed by a `summary-only` route that has been created to aggregate this route
 - `b` backup-path

#### Column 2
 - `>` best path

#### Column 3
 - `i` learned via iBGP
 - `(blank)` learned from eBGP peer, or local router if next hop is `0.0.0.0`

#### Column 4
- `Network`: displayed without prefix length if network/subnet is classful

#### Column 5
- `Next Hop`: `0.0.0.0` = we are the originator

#### Column 6, 7, 8: Path attributes
- `MED`
- `LocPrf`
- `Weight`

#### Column 9
- `ASN`

#### Column 10
- `Origin code`
    - `0` = IGP (network command used to advertise this route)
    - `1` = EGP
    - `2` = Unknown (redistribution command used to advertise this route)