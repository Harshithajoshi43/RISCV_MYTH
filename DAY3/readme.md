
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e9b7852b-bf31-448c-a78a-0f1f46f15b91" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8b12b462-d70d-42a2-8c88-c6bff8f59f14" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/eed2aba3-136e-4843-8958-e040692fc8e1" />
<img width="1914" height="1071" alt="image" src="https://github.com/user-attachments/assets/b1be106f-624d-4edc-b996-4be1021f30eb" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/556ca674-2763-4e08-8d75-aadebe1734d8" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/155a5bc9-1244-408e-b231-2cbcdefb70ca" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ad7c8060-52c0-4518-a12d-50d467e703d6" />
```
\m5_TLV_version 1d: tl-x.org
\m5
   // =================================================
   // Welcome!  New to Makerchip? Try the "Learn" menu.
   // =================================================
   
   //use(m5-1.0)   /// uncomment to use M5 macro library
\SV
   // Macro providing required top-level module definition, random
   // stimulus support, and Verilator config.
   m5_makerchip_module   // (Expanded in Nav-TLV pane.)
\TLV
   |calc
      @1
         $reset = *reset;
         
         // Free-running counter
         $cnt[31:0] = $reset ? 32'b0 : (>>1$cnt + 32'b1);
         
         // Calculator inputs ($val1 fed back from previous $out)
         $val1[31:0] = >>1$out[31:0];
         $val2[31:0] = $rand2[3:0];
         $op[1:0]    = $rand3[1:0];
         
         // Arithmetic operations
         $sum[31:0]  = $val1 + $val2;
         $diff[31:0] = $val1 - $val2;
         $prod[31:0] = $val1 * $val2;
         $quot[31:0] = ($val2 != 0) ? ($val1 / $val2) : 32'hDEAD_BEEF;
         
         // Output multiplexer with reset
         $out[31:0] = $reset          ? 32'b0 :
                      ($op == 2'b00)  ? $sum  :
                      ($op == 2'b01)  ? $diff :
                      ($op == 2'b10)  ? $prod :
                                        $quot;

   // Assert these to end simulation before the cycle limit.
   *passed = *cyc_cnt > 40;
   *failed = 1'b0;
\SV
   endmodule
```

<img width="1918" height="964" alt="image" src="https://github.com/user-attachments/assets/0342699d-5003-4a6a-a904-5daa840732a0" />

