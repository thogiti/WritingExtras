
# Some typos and errors in the paper
[Does Your Blockchain Need Multidimensional Transaction Fees?](https://arxiv.org/abs/2504.15438)

arXiv:2504.15438 [cs.GT] (or arXiv:2504.15438v1 [cs.GT] for this version)  	
https://doi.org/10.48550/arXiv.2504.15438



1. 
   - Location: Section 1, Figure 1 and caption (Page 3).  
   - Issue: The figure references $x \le 30$, $y \le 6$, plus $x + 5y \le 30$ or $x + 6y \le 30$. Meanwhile, the text references 36 for gas and 6 for blobs.  Is this a typo?

2. 
   - Location: Section 2, Definition 2 (Page 5).  
   - Issue: The text shows $\sum_j x_i W_{ij} \le B_j$, it should be $\sum_i x_i W_{ij} \le B_j$.  
   - Correct summation is:
     $$\sum_{i} x_i W_{ij} \le B_{j}.$$

3.
   - Location: Section 5.1, “Practical Bounds” example (Page 9).  
   - Issue: The example claims the maximizing column yields 0.85, leading to $\alpha^{hist} = 20/17$. A correct calculation might yield 0.95 instead, changing $\alpha^{hist} = 1/0.95 = 20/19$.  
   
4. 
   - Location: Section 4.2 (Theorem 4.3) or Appendix (Page 16).  
   - Issue: You may need to state that $A_{i\ell} > 0$ in the utility matrix.  

5.
   - Location: Section 4, Theorem 4 and Corollary 5 (Page 8).  
   - Issue: Typos in “dimensions” and “partition.”  
   
6. 
   - Location: Appendix A.2, Proof of Corollary 5 (Page 17).  
   - Issue: “almong” → “among.”  




