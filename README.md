# Generating a shift from baseline table using multidimensional arrays and PROC FCMP

## Abstract
A shift from baseline table, also known as a shift table, is an n-by-n matrix in which the row and column positions correspond respectively to a category before and after a time point and/or treatment. 

This paper will discuss (1) a way to tally the count data for a shift table using a multidimensional array and (2) a way to calculate percentages for a shift table using matrix operations built into the SAS Function Compiler (PROC FCMP), which (unlike PROC IML) is part of Base SAS.

#

To be presented at a future SAS users conference.


# Illustration: Code for a 4x4 table

```
data scores;
	infile cards;
	input subjid $ visit score;
	cards;
1 0 4
1 1 3
2 0 1
2 1 3
3 0 2
3 1 2
4 0 2
4 1 2
;

proc sql ;
	select count(unique subjid) into :nsubj
		from scores
	;
quit;


data counts;
	set scores end=eof;
	by subjid;
	
	retain prepost1-prepost16 0 _pre _post;
	if first.subjid then call missing(_pre, _post);
	
	array prepost[4,4] prepost1-prepost16;
	if visit = 0 then _pre = score;
	else if visit = 1 then _post = score ;
	
	if last.subjid then do;
		prepost[_pre, _post] = sum(prepost[_pre, _post], 1);
	end;
	
	if eof then do;
		array post[4] post1-post4;
		do i = 1 to 4;
			pre = i;
			do j = 1 to 4;
				post[j] = prepost[i, j];						
			end;
			output;
		end;		

	end;
	keep post: ;
run;

	/******************/
	
proc fcmp ; 
	array counts[4, 4] / nosymbols;
	rc = read_array('counts', counts);
	
	array denom[4, 4] ;
	call fillmatrix(denom, 1/&nsubj);
	put denom=; 
	
	array pcts[4, 4];
	call elemmult(counts, denom, pcts);
		
	rc = write_array('counts', counts);
	rc = write_array('pcts', pcts);
run;

%*options cmplib=work.mylib;  /* not needed if only using pre-defined FCMP functions */


data shifttable;
	format pre post1-post4;
	set counts;
	set pcts;
	
	pre = _n_;
	
	length post1-post4 $20;
	
	array counts[4];
	array pcts[4];
	array post[4] $;
	do i = 1 to 4;
		if counts[i] > 0 and pcts[i] > 0 then post[i] = put(counts[i], 3.) || ' (' || substr(put(pcts[i], percent9.1), 3) || ')';
		else post[i] = put(counts[i], 3.);
	end;
	keep pre post: ;
run;

proc print data=&syslast (obs=20); 
run;

```

#

Posted 2017-08-21

Updated 2017-08-22


#

[SAS Tips](/sas-tips)

[Portfolio](/)

