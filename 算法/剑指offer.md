# 剑指offer

## 位运算

### 二进制中1的个数

左移动一位——>二进制后面加两个0——> 乘二进制的100 ——>乘十进制的2
二进制中1的个数：不是有多少个数，而一眼看过去有多少个1
？？书上P79说什么负数没看懂
减一+与，这个数就会少个1。 那为什么还与呢？当后面位是00的情况，减一会使后面的0变成1，在与后面新生出来的1和第一个1一起消灭。
能消灭多少次，就说明有多少个1.

1111-1=1110&1111=1110(后面是与不与影响不大)
1100-1=1011&1100=1000（这个-1后就要靠与把后面新生成的1消除）

```

  public int NumberOf1(int n) {
        int count=0;
        while(n!=0){
            count++;
              n=(n-1)&n;
        }
         return count;
    }
```



### 不用+号做加法

 先异或再与
先相加在加进制如果还进了就一直加进制 
num2!=0 num2是有可能等于负数的

```

public int Add(int num1,int num2) {
      while (num2!=0) {
            int t = num1 ^ num2;
            num2 = (num1 & num2) << 1;
            num1=t;
        }
        return num1;}


```

### 求1+2+..n 

短路运算做判断
这个递归是从大到小
后面也是一个运算哦

```
public int Sum_Solution(int n) {
      int sum=n;
        boolean r=(sum>0)&&((sum=sum+Sum_Solution(--sum))>0);
        return sum;
    }
```

## 树

### 镜像 

最后没有返回语句也行，做完了就返回到上一层了

```

public class Solution {
    public void Mirror(TreeNode root) {
        if(root==null){
            return;
        }
        TreeNode t=root.left;
        root.left=root.right;
        root.right=t;
            Mirror(root.left);
            Mirror(root.right);
    }}


```



### 序列化反序列化

 标准递归 反加一个函数分割字符串
下划线是为了分割和区分一个节点，#是分辨比如1_1_1没有#可以画出好多种树

```
public class Solution {
    String Serialize(TreeNode root) {
        if(root==null){
            return "#_";
        }
        String str=root.val+"_";
        str+=Serialize(root.left);
        str+=Serialize(root.right);
        return str;
  }
    TreeNode Deserialize(String str) {
       String[] arr=str.split("_");
       Queue<String> queue=new LinkedList<>();
        //for(int i=0;i<arr.length;i++){
        for(String s:arr){
           //queue.offer(arr[i]);
        queue.offer(s);
        }
       return handler(queue);
        
  }
//在队列没有的时候已经往下一步了，到叶子是会返回上一层走，走到上一层从代码来看是往下一行，到最后已经返回了
    TreeNode handler(Queue<String>queue){
        String s=queue.poll();
        if(s.equals("#")){
            return null;
        }
        TreeNode node = new TreeNode(Integer.valueOf(s));
        node.left=handler(queue);
        node.right=handler(queue);
        return node;
    }
}
```

### 对称 

左右要作为参数传入进行比较 比较时候要要先排除为空的情况**坑：1.不是判断左右是否一样，是判断对称****直接比较不完事了吗，咋前面还有先比较是不是为空？答：要是为空比较那不是空指针异常了吗**

```
public class Solution {
    boolean isSymmetrical(TreeNode pRoot)
    { if(pRoot==null){
            return true;
        }
   return com(pRoot.left,pRoot.right);
        
    }
    boolean com(TreeNode left,TreeNode right)
    {
        if(left==null){
            return right==null;
        }//此时left一定不为空，但是right可能为空，还不能比较
        if(right==null)
            return false;
        if(left.val!=right.val)
            return false;
        return com(left.right,right.left)&&com(left.left,right.right);
    }
    
}
```

### 是否是平衡树

 结果类两个条件 左右子树是否是平衡树 左右子树之差是否在1以内

```


public class Solution {
    class Result{
        int h;
        boolean isB;
        Result(int h,boolean isB){
            this.h=h;
            this.isB=isB;
            
        }
    }
    Result isBalance(TreeNode root){
        if(root==null){
            return new Result(0,true);
        }
        Result leftB=isBalance(root.left);
        if(!leftB.isB){
           return new Result(0,false);
        }
        Result rightB=isBalance(root.right);
        if(!rightB.isB){
          return new Result(0,false);
        }
        if(Math.abs(leftB.h-rightB.h)>1){
            return new Result(0,false);
        }
         return new Result(Math.max(leftB.h,rightB.h)+1,true);
    }
    public boolean IsBalanced_Solution(TreeNode root) {
        Result result= isBalance(root);
        return result.isB;
        
    }
}
```

### 树的深度

```
public int TreeDepth(TreeNode root) {
        if(root==null){
            return 0;
        }
        int leftdep=TreeDepth(root.left);
        int rightdep=TreeDepth(root.right);
      
        return leftdep>rightdep?leftdep+1:rightdep+1;
        
    }
```

### 重建二叉树 

四个参数
不管递归过程是什么样的，先序序列长度和中序序列长度一定要相同

而中序序列的边界是比较好确定的，左子树中中序序列起点是startIn,终点是i-1

左子树先序序列起点是startPre+1,要求左子树先序序列终点，假设该终点下标为x,

根据中序序列长度和先序序列长度相等的条件

得x-(startPre+1)=i-1-startIn>>x-startPre=i-startIn>>x=i-startIn+startPre,

也就是说，左子树先序的终点等于起点加长度=(startPre+1)+(i-1-starIn) 

左子树的长度 是总长度减中序的起点减一

```
public class Solution {
    public TreeNode reConstructBinaryTree(int [] pre,int [] in) {
       return re(pre,0,pre.length-1,in,0,in.length-1);
    }
   public TreeNode re(int [] pre,int starPre,int endPre,int [] in,int starIn,int endIn){
     if(starPre>endPre||starIn>endIn){
         return null;
     }
       int rootVal= pre[starPre];
       TreeNode root=new TreeNode(rootVal);
       for(int i=0;i<=endIn;i++){
           if(in[i]==r    ootVal){
            root.left=re(pre,starPre+1,starPre+i-starIn,in,starIn,i-1);
            root.right=re(pre,starPre+1+i-starIn,endPre,in,i+1,endIn);
           }
       }
      return root;
   }
}
```

### 树的子结构 

**两个递归**

第一个找1树中有没有2树根这个值

第二个找1树中以R为根节点的子树是否和B树有相同的结构

能兜一种是一种

```
public class Solution {
    //找1树中有没有2树根这个值
    public boolean HasSubtree(TreeNode root1,TreeNode root2) {
        boolean r=false;
      if(root1!=null&&root2!=null){
          if(root1.val==root2.val)r=f(root1,root2);
          if(!r)r=HasSubtree(root1.left,root2);
          if(!r)r=HasSubtree(root1.right,root2);
      }
        return r;
    }
     public boolean f(TreeNode root1,TreeNode root2) {
         //有可能出现1=null但2也=null的情况
        //如果Tree2还没有遍历完，Tree1却遍历完了。返回false 那tree没有遍历完的肯定是多余的
         if(root1==null&& root2 != null)return false;
//如果Tree2已经遍历完了都能对应的上，返回true
         if(root2==null)return true;
         if(root1.val==root2.val)
             return f(root1.left,root2.left)&&f(root1.right,root2.right);
         return false;
     }
}
```

### 某二叉搜索树的后序遍历

 两个for找查左右子树

遍历找到第一个比根大的，后面的都不能比根大了

难点在于，以第一个比根大的节点划分为两段做递归，直到边界说明遍历完成，没有找到不合格的地方，成功了

```
public static boolean VerifySquenceOfBST(int[] sequence) {
//只需要判断右子树中是否有比root高的，从0开始也无所谓因为前面一定都小
        if (sequence.length <= 0 || sequence == null)
            return false;
        return ju(sequence, 0, sequence.length - 1);
    }
    static boolean ju(int[] sequence, int star, int root) {
        if ( root <= star) {
            return true;
        }int i = star;
        for (; i < root; i++) {
            if (sequence[i] > sequence[root])
                break;
        }
        for (int j = i; j < root; j++) {
            if (sequence[j] < sequence[root])
                return false;
        }
        return ju(sequence, star, i - 1) && ju(sequence, i, root - 1);
    }
```

## 字符串

### 翻转单词顺序列

分割成字符数组再倒序append，注意空格

```


public String ReverseSentence(String str) {
if (str.trim().equals("")) {
            return str;
        }
        String[] sarr = str.split(" ");
        StringBuilder sb = new StringBuilder();
        for (int i = sarr.length - 1; i >= 0; i--) {
            if (i == 0) {
                sb.append(sarr[i]);
            } else {
                sb.append(sarr[i] + " ");
            }
        }
        return sb.toString();
        }

```

### 左旋转字符串

注意：
str.substring(0, n);//取的到前取不到后的！
n>=str.length()如果长度为3，n也为3，不是直接返回“”了吗

```

public String LeftRotateString(String str,int n) {
if (str.trim().equals("") || n > str.length())
    return "";
String appendEnd = str.substring(0, n);//取的到前取不到后的！
String appendStart = str.substring(n, str.length());
return appendStart += appendEnd;
    }
```



```

```





## 数组

### 超过一半的数

开心消消乐

```
public static int MoreThanHalfNum_Solution(int[] array) {
    if (array==null||array.length<1)
        return 0;
    int count = 1;//进入循环就不能做初始化了，因为得1之后马上要减掉，循环的if只是为了不减到负数
    int same = array[0];
    for (int i = 1; i < array.length; i++) {
        if (array[i] == same) {
            count++;
        } else {
            if (count == 0) {//没有这个 count会被减到负，有了不相等就一直保持0
                same = array[i];
                count = 1;
            }
            count--;
        }
    }
    return count==0?0:same;
}
```





```
public class Solution {
        public  int GetNumberOfK(int[] array,int k){
        if(array==null||array.length==0)
            return 0;
        int first=getFirstK(array,k,0,array.length-1);
        int last=getLastK(array,k,0,array.length-1);
        if(first==-1 ||last==-1){
            return 0;
        }
        else{
            return last-first+1;
        }
         
    }
     
    public  int getFirstK(int[] array,int k,int start,int end){
        while(start<=end){
            int mid=(start+end)/2;
            if(k<array[mid])
                end=mid-1;
            else if(k>array[mid])
                start=mid+1;
            else{
//寻找头，要么是第一个位置的元素，要么前面没有重复的
                if((mid>0&&array[mid-1]!=k)||mid==0)
                    return mid;
                else{
//如果k等于mid，但是前面有重复的，那么就在前面找，后下标往前移动，缩小范围。
                    end=mid-1;
                }
            }
        }
        return -1;
    }
     
    public  int getLastK(int[] array,int k ,int start,int end){
        while(start<=end){
            int mid=(start+end)/2;
            if(k<array[mid])
                end=mid-1;
            else if(k>array[mid])
                start=mid+1;
            else{
//寻找尾要么就是最后一个位置，要么就是后面没有重复的
                if((mid<array.length-1&&array[mid+1]!=k)||mid==array.length-1)
                    return mid;
                else{
                    start=mid+1;
                }
            }
        }
        return -1;
    }
}
```

### 奇数放前，偶数放后

```
   public void reOrderArray(int [] array) {
        //前面不能为偶，后面不能为奇
        int star=0;
        int end=array.length-1;
        while(star<end){//不要相等有可能会超过
            if(array[star]%2==0&&array[end]%2!=0){
                int t=array[star];
                array[star]=array[end];
                array[end]=t;
             }else if(array[end]%2==0){
                end--;
            }else if(array[star]%2!=0){
                star++;
        }
        }
    }
```

进阶问题：位置不能变

```
[1,2,3,4,5,6,7]
对应输出应该为:
[1,3,5,7,2,4,6]
你的输出为:
[1,7,3,5,4,6,2]
```



在数组中查找两个数，使得他们的和正好是S，如果有多对数字的和等于S，输出两个数的乘积最小的。**两个头尾指针往中间移动，只能e--或s++**
**应该是有序的吧****？？**   



```
public ArrayList<Integer> FindNumbersWithSum(int [] array,int sum) {
         ArrayList<Integer> list=new  ArrayList<Integer>();
        if(array.length<=1)
            return list;
        int s=0,e=array.length-1;
        while(s<e){
           int add= array[s]+array[e];
            if(sum==add){
                list.add(array[s]);
                list.add(array[e]);
                return list;//没有这句会运行超时
            }else if(add>sum){
                e--;
            }else{
                s++;
            }
        }
          return list;
    }
```



### 连续子数组最大和 

好好想想第三步加上数字3，此时由于之前的累计和是-1，小于0，那么用-1加上3，比3本身还要小。也就是说从第一个数字开始的累计和会小于第三个数字开始的子数组的和。因为之前的累计和被抛弃max的作用是防止 后面是一个负数加上有损利益，但整体和并没有小于负数。比如当sum=18，max=18，下一个数是-5.如果没有max加入去之后就忘记之前的中间可能会出现的最大值。

```
public static int FindGreatestSumOfSubArray(int[] array) {
    if (array.length == 0)
        return 0;
    int sum = array[0], max = array[0];
    for (int i = 1; i < array.length; i++) {
        if (sum >= 0)////之前的和小于零，还不如直接用下一个
            sum += array[i];
        else
            sum = array[i];
        if (sum > max) {//不断刷新最大值记录，防止后面是一个负数
            max = sum;
        }
    }
    return max;
}
```

动态规划版本没太看懂

```

public  int FindGreatestSumOfSubArray(int[] array) {
        int res = array[0]; //记录当前所有子数组的和的最大值
        int max=array[0];   //包含array[i]的连续数组最大值
        for (int i = 1; i < array.length; i++) {
            max=Math.max(max+array[i], array[i]);
            res=Math.max(max, res);
        }
        return res;
}
```



### 把数组排成最小的数

不可以用数组这样做str1和str2是Integer，会先相加后再与空字符串拼接
**把数组里的数进行拼接起来谁大的排序 ，随和把数组遍历加成一个字符串**

```
public String PrintMinNumber(int[] numbers) {
    int n;
    String s = "";
    ArrayList<Integer> list = new ArrayList<Integer>();
    n = numbers.length;
    for (int i = 0; i < n; i++) {
        list.add(numbers[i]);
    }
    Collections.sort(list, new Comparator<Integer>() {
        public int compare(Integer str1, Integer str2) {
            String s1 = str1 + "" + str2;
            String s2 = str2 + "" + str1;
// //当然是以调用者为标准，调用者更小返回负数
            return s1.compareTo(s2);
        }
    });
    for (int j : list) {
        s += j;
    }
    return s;
}
```



### 构建乘积数组

f数组存之前的结果

b数组存之后的结果

B[i]=f[i]*b[i]

```
  public int[] multiply(int[] A) {
        int length =A.length;
        int []f=new int[length];
        int []b=new int[length];
        int []B=new int[length];
        f[0]=1;
        b[0]=1;
        for(int i=1;i<length;i++){
           //之前的乘积数和之后的乘积数都不包括本来的数，所以要乘上
            f[i] = A[i - 1]*forword[i-1];//数组的前一个元素乘上之前的乘积
            b[i]=A[length-i]*b[i-1];
        }
        for(int j=0;j<length;j++){
            B[j]=f[j]*b[len - j -1];
        }
        return B;
    }

```

### 查找旋转数组的最小数 

二分



采用二分法解答这个问题，
mid = low + (high - low)/2
需要考虑三种情况：
(1)array[mid] > array[high]:
出现这种情况的array类似[3,4,5,6,0,1,2]，此时最小数字一定在mid的右边。
low = mid + 1
(2)array[mid] == array[high]:
出现这种情况的array类似 [1,0,1,1,1] 或者[1,1,1,0,1]，此时最小数字不好判断在mid左边
还是右边,这时只好一个一个试 ，
high = high - 1
(3)array[mid] < array[high]:
出现这种情况的array类似[2,2,3,4,5,6,6],此时最小数字一定就是array[mid]或者在mid的左
边。因为右边必然都是递增的。
high = mid
注意这里有个坑：如果待查询的范围最后只剩两个数，那么mid 一定会指向下标靠前的数字
比如 array = [4,6]
array[low] = 4 ;array[mid] = 4 ; array[high] = 6 ;
如果high = mid - 1，就会产生错误， 因此high = mid
但情形(1)中low = mid + 1就不会错误

## 矩阵



二维数组的length就是有多少行，想知道多少列： matrix[0].length
row是列，col是行
先横后竖，array[matrix.length][matrix[0].length]

### 顺时针打印矩阵

行不动列加动，行加动列不动
行不动列减动，行减动列不动
一开始坐标是两个sta,第一个循环：hangsta到hangend。第二循环：liesta到lieend
到了右下角从两个end开始，第一个循环从hangend到hangsta。第二循环：从lieend到liesta
  if (hangSta != hangEnd)如果不加这个会把中间的值再打印一遍



```

import java.util.ArrayList;
public class Solution {
public ArrayList<Integer> printMatrix(int[][] matrix) {
        ArrayList<Integer> list = new ArrayList<Integer>();
        if (matrix == null || matrix.length < 1) {
            return list;
        }
        int lieSta = 0;
        int hangSta = 0;
        int hangEnd = matrix.length - 1;
        int lieEend = matrix[0].length - 1;
        // 打印上边一条边
        //一条横的
        if (hangSta == lieEend) {
            for (int i = 0; i < matrix.length; i++) {
                list.add(matrix[i][hangSta]);
            }
            return list;
        }        //一条竖的
        if (lieSta == hangEnd) {
            for (int i = 0; i < matrix[0].length; i++) {
                list.add(matrix[hangEnd][i]);
            }
            return list;
        }
        while (lieSta <= lieEend && hangEnd >= hangSta) {
            int hang = lieSta, lie = hangSta;
            for (lie = lieSta; lie <= lieEend; lie++) {
                list.add(matrix[hang][lie]);
            }
            lie--;
            for (hang = hangSta+1; hang <= hangEnd; hang++) {
                list.add(matrix[hang][lie]);
            }
            hang--;
         if (hangSta != hangEnd)
                for (lie = lieEend - 1; lie >= lieSta; lie--) {
                    list.add(matrix[hang][lie]);
                  
                }
            lie++;
            if (lieSta != lieEend)
                for (hang = hangEnd - 1; hang > hangSta; hang--) {
                    list.add(matrix[hang][lie]);
           
                }
            lieSta++;
           hangSta++;
            lieEend--;
            hangEnd--;
        }
        return list;
    }
}



```

### 矩阵0所在的行列都设置为0

给定一个 m x n 的矩阵，如果一个元素为 0，则将其所在行和列的所有元素都设为 0。请使用原地算法。

```


    public void setZeroes(int[][] matrix) {
        int hang = matrix.length;
        int lie = matrix[0].length;
        int[] hArr = new int[hang];
        int[] lArr = new int[lie];
        for (int i = 0; i < hang; i++) {
            for (int j = 0; j < lie; j++) {
                if (matrix[i][j] == 0) {
                    hArr[i] = 1;
                    lArr[j] = 1;
                }}}
        for (int i = 0; i < hang; i++) {
            for (int j = 0; j < lie; j++) {
                if (hArr[i] == 1 || lArr[j] == 1) {
                    matrix[i][j] = 0;
                }}}
 }
```



### 在递增数组中判断是否有一个数

 while 三if左下移动


```

    public boolean Find(int target, int [][] array) {
        int l=0;
        int h=array.length-1;
        while(l<array[0].length&&h>=0){
            if(array[l][h]==target){
                return true;
            }else if(array[l][h]>target){
               h--;
            }else{
                l++;
            }
        }
        return false;}
```

