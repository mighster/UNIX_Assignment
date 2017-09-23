Kevin Falk
UNIX Assignment
9/22/2017

#Data Inspection:
#Before starting, good practices is to check to ensure the files are
#Input
for filename in snp_position.txt fang_et_al_genotypes.txt; 
do echo "File size of $filename: $(du -h)";
echo "Total columns of $filename: $(awk -F "\t" '{print NF; exit}' $filename)"; 
echo "Total lines: $(wc -l $filename)"; 
echo "ASCII check:" $(file $filename); 
done

#Output
#File size of snp_position.txt: 14G      .
#Total columns of snp_position.txt: 15
#Total lines: 983 snp_position.txt
#ASCII check: snp_position.txt: ASCII text, with CRLF line terminators
#File size of fang_et_al_genotypes.txt: 14G      .
#Total columns of fang_et_al_genotypes.txt: 986
#Total lines: 2783 fang_et_al_genotypes.txt
#ASCII check: fang_et_al_genotypes.txt: ASCII text, with very long lines


#Data Processing
#divide the original fang_et_al_genotypes file into two, one for corn, one for teosinte
grep -E "(ZMMIL|ZMMLR|ZMMMR)" fang_et_al_genotypes.txt > corn.txt
grep -E "(ZMPBA|ZMPIL|ZMPJA)" fang_et_al_genotypes.txt > teosinte.txt

#remove header
grep "Group" fang_et_al_genotypes.txt > headeronly.txt

#add header back onto each individual file
cat headeronly.txt corn.txt > corn_w_header.txt 
cat headeronly.txt teosinte.txt > teosinte_w_header.txt 

#snp ID is the common column between fang_et_al_genotypes and snp_position files. 
#In order to join the files we need to remove the first two (Sample_ID and JG_OTU) columns
cut -f 3-968 corn_w_header.txt > corn_2.txt 
cut -f 3-968 teosinte_w_header.txt > teosinte_2.txt

#Transpose data so the columns in both files align using the snp_ID
awk -f transpose.awk corn_2.txt > transposed_corn.txt
awk -f transpose.awk teosinte_2.txt > transposed_teosinte.txt

#Create new file using snp_position.txt with only columns 1, 3 and 4 (snp ID, Chromosome number and snp position in the Genome)
grep -v "^#" snp_position.txt | cut -f 1,3,4 > snp_position_new.txt

#Remove headers from transposed files
grep -v "Group" transposed_corn.txt > corn_no_header.txt
grep -v "Group" transposed_teosinte.txt > teosinte_no_header.txt 
grep -v "SNP_ID" snp_position.txt > snp_no_header.txt

#Sort headerless corn, teosinte genotype and snp files by their snp_ID (first column)
sort -k1,1 snp_no_header.txt > snp_sorted.txt 
sort -k1,1 corn_no_header.txt > corn_sorted.txt
sort -k1,1 teosinte_no_header.txt > teosinte_sorted.txt

#Check for errors in the sorting process
for filename in snp_sorted.txt corn_sorted.txt teosinte_sorted.txt; do sort -k1,1 $filename | echo $? $filename; done

#Output:
#0 snp_sorted.txt
#0 corn_sorted.txt
#0 teosinte_sorted.txt

#Join snp and genotype files by their snp_ID coloumns
join -t $'\t' -1 1 -2 1 snp_sorted.txt corn_sorted.txt > join_corn.txt
join -t $'\t' -1 1 -2 1 snp_sorted.txt teosinte_sorted.txt > join_teosinte.txt

#Sort files by chromosome number
sort -k2,2n join_corn.txt > chromosome_sort_corn.txt 
sort -k2,2n join_teosinte.txt > chromosome_sort_teosinte.txt

#Substitute missing data (?) with '-'
sed 's/?/-/g' chromosome_sort_corn.txt > substituted_corn.txt
sed 's/?/-/g' chromosome_sort_teosinte.txt > substituted_teosinte.txt
	
#For loops to split into 10 chromosome files and sort for the increasing (?) files
for i in {1..10}; do awk '$2=='$i'' chromosome_sort_corn.txt > corn_chr"$i".txt; done
for i in {1..10}; do awk '$2=='$i'' chromosome_sort_teosinte.txt > teosinte_chr"$i".txt; done

#For loop to sort for the increasing (?) files
for i in {1..10}; do sort -k3,3n corn_chr"$i".txt > corn_chr"$i"_increasing_ordered.txt; done
for i in {1..10}; do sort -k3,3n teosinte_chr"$i".txt > teosinte_chr"$i"_increasing_ordered.txt; done

#For loops to split into 10 chromosome files for the decreasing (-) files
for i in {1..10}; do awk '$2=='$i'' substituted_corn.txt > corn_chr"$i"_decreasing.txt; done
for i in {1..10}; do awk '$2=='$i'' substituted_teosinte.txt > teosinte_chr"$i"_decreasing.txt; done

for i in {1..10}; do sort -k3,3n corn_chr"$i"_decreasing.txt > corn_chr"$i"_decreasing_ordered.txt; done
for i in {1..10}; do sort -k3,3n teosinte_chr"$i"_decreasing.txt > teosinte_chr"$i"_decreasing_ordered.txt; done

#File with all snps with unknown positions in the genome
grep "unknown" chromosome_sort_corn.txt > unknown_corn.txt
grep "unknown" chromosome_sort_teosinte.txt  > unknown_teosinte.txt

#File with all snps with multiple positions in the genome
grep "multiple" chromosome_sort_corn.txt > multiple_corn.txt
grep "multiple" chromosome_sort_teosinte.txt > multiple_teosinte.txt


#Commit and push files to github
git init
git add .
git add README.md 
git commit -m "Kevin's assignment" 
git push origin master

