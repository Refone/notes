# Intel 手册常用信息

## Extended-Page-Table Pointer (EPTP)
> 24.6.11

| Bit Position(s) | Field |
|:---|:---|
| 2:0 | EPT paging-structure memory type (see Section 28.2.6): <br> 0 = Uncacheable (UC) <br> 6 = Write-back (WB) <br> Other values are reserved. <br> |
| 5:3 | This value is 1 less than the EPT page-walk length (see Section 28.2.2) |
| 6 | Setting this control to 1 enables accessed and dirty flags for EPT (see Section 28.2.4) |
| 11:7 | Reserved |
|(N-1):12 | Bits N–1:12 of the physical address of the 4-KByte aligned EPT PML4 table |
| 63:N | Reserved |



