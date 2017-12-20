## 红黑树（Red-Black Tree）解析
基于Java的红黑树结构及操作解析

这一篇我们来聊聊红黑树，写这篇文章的起因是在阅读HashMap源码时，发现JDK1.8对于HashMap的实现引入了红黑树来处理哈希冲突以提高性能，而红黑树的数据结构和操作都是较为复杂的，自己看得过程中有些地方也反复了多次。。。俗话说得好，好记性不如烂笔头，因此决定写下这篇笔记供自己和需要的人日后参考。在开始之前，首先要感谢**张拭心**同学的这两篇关于红黑树和二叉查找树的文章：

http://blog.csdn.net/u011240877/article/details/53242179  

http://blog.csdn.net/u011240877/article/details/53329023

这两篇文章讲得十分详细，使我受益匪浅，在这里也强烈推荐大家阅读一下。由于拭心同学的文章在分析二叉查找树的查找，插入和删除时引用的是递归的实现，为了不重复，本文分析时将采用循环的实现，为大家提供另一种思路。

OK，正式开始，何为红黑树？**红黑树（Red-Black Tree）** 是一种自平衡二叉查找树，其每个节点都带有黑或红的颜色属性。由于它的本质也是一种二叉查找树，因此它的查找，插入和删除操作均以二叉查找树的对应操作作为基础；但由于红黑树自身要保证平衡（也即要始终满足其五条特性，这个下文会有详述），每次插入和删除之后它都要进行额外的调整，以恢复自身的平衡，这是它与普通二叉查找树不同的地方，也正因为如此，红黑树的查找，插入和删除操作在最坏情况下的时间复杂度也能保证为O(logN)，其中N为树中元素个数。

既然红黑树本质是二叉查找树，那么就有必要先来看一下二叉查找树的相关知识。

- **二叉查找树**

	**二叉查找树（Binary Search Tree）**，又名二叉排序树，二叉搜索树，B树。顾名思义，它的节点是可比较的并且具有以下性质：
	
	a. 若左子树不为空，则根节点的值大于其所有左子树中节点的值；  
	b. 若右子树不为空，则根节点的值小于或等于其所有右子树中节点的值；  
	c. 左右子树也分别为二叉查找树；	
	d. 没有键值相等的节点。
	
	由于以上性质，中序遍历二叉查找树可得到一个关键字的有序序列，一个无序序列可以通过构造一棵二叉查找树变成一个有序序列，构造树的过程即为对无序序列进行查找的过程。每次插入的新的结点都是二叉查找树上新的叶子结点，在进行插入操作时，不必移动其它结点，只需改动某个结点的指针，由空变为非空即可。搜索、插入、删除的复杂度等于树高，期望 O(logN)，最坏 O(N)（数列有序，树退化成线性表）。
	
	这里先给出一个二叉查找树节点的结构，下文代码中就用它作为树节点的类：
	
	```
	class BSTNode{
		int value;  //节点的值
		BSTNode left;  //节点的左子树
		BSTNode right;  //节点的右子树
		BSTNode parent;  //节点的父节点
			
        BSTNode(int value, BSTNode parent) {
            this.value = value;
            this.parent = parent;
        }
        
        @Override
        public boolean equals(Object obj)
        {
        	  //两个节点的value相等，则认为两个节点相等
            return (obj instanceof BSTNode) && (((BSTNode) obj).value == this.value);
        }
	}
	```
	
	下面就分别看一下二叉查找树的查找，插入和删除操作的实现，此处采用循环来实现。

	- **查找**

		在二叉搜索树T中查找key的过程为：

		a. 若T是空树，则搜索失败，否则：	
		b. 若key等于T的根节点的数据域之值，则查找成功；否则：  
		c. 若key小于T的根节点的数据域之值，则搜索左子树；否则：  
		d. 查找右子树。
		
		下面是这个过程的Java代码实现：
		
		```
		/**
     	* @param key 目标节点的键值
     	* @return 与key匹配的节点，若未能成功匹配则返回null
     	*/
    	BSTNode searchBST(int key) {
    		//若根节点为空，或根节点与key匹配成功，则直接返回
        	if (mRoot == null || mRoot.value == key) {
            	return mRoot;
        	}
        
        	BSTNode t = mRoot;
        		
        	//从根节点开始循环查找
        	do {
            	if (key < t.value) {
                	t = t.left; //若key比节点小，则在左子树中继续查找
            	}
            	else if (key > t.value) {
                	t = t.right; //若key比节点大，则在右子树中继续查找
            	}
            	else {
                	return t; //匹配成功，返回匹配节点
            	}
        	}
        	while (t != null);
       
        	return null; //匹配失败，返回null
    	}  
		```
	- **插入**

		插入可以理解为先查找，找到了就说明已经存在该节点不用再进行插入了（也有可能找到后做覆盖操作，比如HashMap的put方法），找不到就将指针最后停留的叶子节点当做待插入节点的父节点，根据两个节点值的大小关系确定该作为左子树还是右子树插入。下面是相关代码：
		
		```
		/**
     	* @param key 待插入节点的键值
     	*/
    	void insertBST(int key) {
        	if (mRoot == null) {
        		//若根节点为空，则使用key创建根节点，插入完成
            	mRoot = new BSTNode(key, null);
            	return;
        	}

        	BSTNode t = mRoot;
        	BSTNode parent; //指向当前遍历到的节点的指针

			//从根节点开始循环查找
        	do {
            	parent = t;

            	if (key < t.value) {
                	t = t.left; //若key比节点小，则在左子树中继续查找
            	}
            	else if (key > t.value) {
                	t = t.right; //若key比节点大，则在右子树中继续查找
            	}
            	else {
                	return; //若key与节点的值相等，则说明节点已存在，不需要插入，直接返回（若需要覆盖节点，在这里完成）
            	}
        	}
        	while (t != null);

			//执行到这一步说明值为key的节点不存在，新创建一个节点，将parent指针指向的节点作为父节点
        	BSTNode nodeToInsert = new BSTNode(key, parent);

        	if (key < parent.value) {
            	parent.left = nodeToInsert; //若key比parent的值小，则作为parent的左子树插入
        	}
        	else {
            	parent.right = nodeToInsert; //若key比parent的值大，则作为parent的右子树插入
        	}
    	}
		```

	- **删除**

		删除操作第一步也是查找，找到待删除节点后分下列几种情况：
		
		a. 若节点为子节点，直接删除即可；	
		
		b. 若节点**只有**左子树或右子树，则删除该节点后，将其**唯一**的子树与父节点相连；
		
		c. 若节点有两个子树，则需要选择一个子树，并从中选出合适的节点K与待删除节点的父节点相连。这时树的结构会发生变化，节点K将接替待删除节点作为这一棵子树的根，那么显然，K需要大于其左子树的所有节点且小于右子树的所有节点。这里对于K有两种选择，要么选择待删除节点的左子树中最大的节点，要么选择其右子树中最小的节点，二者皆可，我们选择前者来实现。下面是删除操作相关代码：
		
		```
		/**
     	* @param key 待删除节点的键值
     	*/
    	void deleteBST(int key) {
        	if (mRoot == null) {
            	return; //若树为空，则无法删除，返回
        	}

        	BSTNode t = mRoot;
        	BSTNode nodeToDelete = null; //需要删除的节点

			//循环查找待删除节点
        	do {
            	if (key < t.value) {
                	t = t.left; //在左子树中继续
            	}
            	else if (key > t.value) {
                	t = t.right; //在右子树中继续
            	}
            	else {
                	nodeToDelete = t; //匹配成功，找到待删除节点，退出循环
                	break;
            	}
        	}
        	while (t != null);

        	if (nodeToDelete == null) {
            	return; //未找到待删除节点，结束返回
        	}

			//若待删除节点为叶子节点
        	if (nodeToDelete.left == null && nodeToDelete.right == null) {
            	if (nodeToDelete.parent == null) {
                mRoot = null; //待删除节点为根节点，置空全局变量mRoot
            }
            else if (nodeToDelete.value < nodeToDelete.parent.value) {
                nodeToDelete.parent.left = null; //待删除节点为其父节点的左子树，则将其父节点的左子树置空
            }
            else {
                nodeToDelete.parent.right = null; //待删除节点为其父节点的右子树，则将其父节点的右子树置空
            }
        	}
        	//若待删除节点有且仅有左子树，则其左子树应该直接接替其位置
        	else if (nodeToDelete.left != null && nodeToDelete.right == null) {
            	if (nodeToDelete.parent == null) {
                	mRoot = nodeToDelete.left; //待删除节点为根节点，则将mRoot赋值为待删除节点的左子树
            	}
            	else if (nodeToDelete.value < nodeToDelete.parent.value) {
                	nodeToDelete.parent.left = nodeToDelete.left; //待删除节点为其父节点的左子树，则将其父节点的左子树赋值为待删除节点的左子树
            	}
            	else {
                	nodeToDelete.parent.right = nodeToDelete.left; //待删除节点为其父节点的右子树，则将其父节点的右子树赋值为待删除节点的左子树
            	}
        	}
        	//若待删除节点有且仅有右子树，则其右子树应该直接接替其位置
        	else if (nodeToDelete.left == null) {
            	if (nodeToDelete.parent == null) {
                	mRoot = nodeToDelete.right; //待删除节点为根节点，则将mRoot赋值为待删除节点的右子树
            	}
            	else if (nodeToDelete.value < nodeToDelete.parent.value) {
                	nodeToDelete.parent.left = nodeToDelete.right; //待删除节点为其父节点的左子树，则将其父节点的左子树赋值为待删除节点的右子树
            	}
            	else {
                	nodeToDelete.parent.right = nodeToDelete.right; //待删除节点为其父节点的右子树，则将其父节点的右子树赋值为待删除节点的右子树
            	}
        	}
        	//若待删除节点的左右子树都存在，则按照上文所说选择一种方案，这里我们选择用其左子树中最大的节点来接替其位置
        	else {
        		/****这里分两步做：
        		 *第一步先组装继承节点；
        		 *第二步将调整好的继承节点与待删除节点的父节点相连。
        		 ****/
        		 
        		/****---第一步---****/
            	BSTNode inheritNode = nodeToDelete.left; //继承节点inheritNode，初始值为待删除节点的左孩子

            	if (inheritNode.right == null) {
            		//若inheritNode没有右子树，则直接上位，将待删除节点（即当前inheritNode的父节点）的右子树据为己有
                	inheritNode.right = nodeToDelete.right;
            	}
            	else {
            		//否则的话找到其左子树中最右边的节点，即左子树中的最大节点
                	while (inheritNode.right != null) {
                    	inheritNode = inheritNode.right;
                	}
                	//到这一步，inheritNode已经是待删除节点左子树中的最大节点
                	inheritNode.parent.right = inheritNode.left; //inheritNode的左子树（可能为空）交给它的父节点
                	inheritNode.left = nodeToDelete.left; //inheritNode上位继承，将其左子树置为待删除节点的左子树
                	inheritNode.right = nodeToDelete.right; //同样，其右子树置为待删除节点的右子树
            	}
            	
            	/****---第二步---****/
            	//到这里继承节点已经调整好了，开始认爸爸
            	if (nodeToDelete.parent == null) {
                	mRoot = inheritNode; //若待删除节点是根节点，则将mRoot置为继承节点inheritNode
            	}
            	else if (nodeToDelete.value < nodeToDelete.parent.value) {
                	nodeToDelete.parent.left = inheritNode; //待删除节点为其父节点的左子树，则将其父节点的左子树赋值为继承节点inheritNode
            	}
            	else {
                	nodeToDelete.parent.right = inheritNode; //待删除节点为其父节点的右子树，则将其父节点的右子树赋值为继承节点inheritNode
            	}
        	}

        	nodeToDelete = null; //将临时变量置空，结束
    	}
		```
		至此，二叉查找树的查找，插入和删除的实现就都分析完毕了。可以看到其性能跟树的结构息息相关，最差情况如果元素有序插入，则会形成一条链表而非树，这时操作的复杂度就变成了O(N)。这个问题在红黑树中得到了很好的解决，下面我们就在二叉查找树的基础上继续分析红黑树。

- **红黑树**

	上文提到过，红黑树是每个节点都带有颜色属性（红或黑）的二叉查找树。在二叉查找树性质的基础上，红黑树额外规定了以下**五条性质**。这些约束确保了红黑树的关键特性：从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。结果是这个树大致上是平衡的。因为操作比如插入、删除和查找某个值的最坏情况时间都要求与树的高度成比例，这个在高度上的理论上限允许红黑树在最坏情况下都是高效的，而不同于普通的二叉查找树。
	
	要知道为什么这些性质确保了这个结果，注意到性质4导致了路径不能有两个毗连的红色节点就足够了。最短的可能路径都是黑色节点，最长的可能路径有交替的红色和黑色节点。因为根据性质5所有最长的路径都有相同数目的黑色节点，这就表明了没有路径能多于任何其他路径的两倍长。
	
	红黑树的查找操作与普通二叉查找树的完全相同，而在进行插入和删除时则有可能导致其不再满足红黑树的性质，因此在这种情况下需要通过节点颜色变更和不超过三次的节点旋转（包括**左旋和右旋**，对于插入操作最多旋转两次）来使其恢复平衡，操作复杂度为O(logN)。下面一一来分析。

	- **五条性质**
	
		a. 每个结点非黑即红；  
		b. 根结点是黑的；  
		c. 每个叶结点（叶结点即指树尾端NIL指针或NULL结点）都是黑的；  
		d. 如果一个结点是红的，那么它的两个儿子都是黑的；  
		e. 对于任一结点而言，其到叶结点树尾端NIL指针的每一条路径都包含相同数目的黑结点。
		
		只有满足这五条性质的二叉查找树才是一棵红黑树，那么当有插入删除操作导致树的平衡被破坏时，就需要通过下面的节点左右旋转操作来重新保证这五条性质，从而恢复红黑树的平衡。

	- **左旋**

		首先说明的是，左旋或右旋都是针对一个节点的操作，而非以整棵树为对象。
		
		先来看对节点X的左旋，X是红黑树中的一个节点，假设其左右孩子都存在，并且左孩子为T，右孩子为Z，即X.left=T, X.right=Z。那么对X左旋的过程为：
		
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/left_rotate.png)
		
		a. 将X变为其右孩子Z的左孩子，即Z替代X成为这一部分树的根节点；		
		b. 将Z原本的左孩子（可能为空）变为X的新右孩子；	
		c. 让Z与X原先的父节点相认（若X原先没有父节点，则Z成为整颗红黑树的新的根节点）。
		
		下面是代码实现：
		
		```
		void rotateLeftBST(BSTNode x) {
			//判断x合法性，若为空或没有右孩子，直接返回
			if (x == null || x.right == null) return;

			//暂存x的父节点parent，右孩子z，以及右孩子z的左孩子zLeft
        	BSTNode parent = x.parent;
        	BSTNode z = x.right;
        	BSTNode zLeft = z.left;

			//上面的步骤a，x变为右孩子z的左孩子
        	z.left = x;
        	x.parent = z;

			//步骤b，z原本的左孩子zLeft变为x的新右孩子
        	x.right = zLeft;
        	if (zLeft != null) {
        		//若zLeft存在，则将x认作新的父亲
            	zLeft.parent = x;
        	}

			//步骤c，z的父节点变为原先x的父节点
        	z.parent = parent;
        	if (parent == null) {
        		//若之前x为整棵树的根节点，则z接替它成为新的根节点mRoot
            	mRoot = z;
        	}
        	//z接替x成为x原先父节点的左孩子或右孩子
        	else if (parent.left == x) {
            	parent.left = z;
        	}
        	else if (parent.right == x) {
            	parent.right = z;
        	}
    	}
		```

	- **右旋**

		下面来看右旋的过程：
		
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/right_rotate.png)
		
		a. 将X变为其左孩子T的右孩子，即T替代X成为这一部分树的根节点；		
		b. 将T原本的右孩子（可能为空）变为X的新左孩子；	
		c. 让T与X原先的父节点相认（若X原先没有父节点，则T成为整颗红黑树的新的根节点）。
		
		下面是代码实现：
		
		```
		void rotateRightBST(BSTNode x) {
			//判断x合法性，若为空或没有左孩子，直接返回
        	if (x == null || x.left == null) return;
			
			//暂存x的父节点parent，左孩子t，以及左孩子t的右孩子tRight
        	BSTNode parent = x.parent;
        	BSTNode t = x.left;
        	BSTNode tRight = t.right;

			//上面的步骤a，x变为左孩子t的右孩子
        	t.right = x;
        	x.parent = t;

			//步骤b，t原本的右孩子tRight变为x的新左孩子
        	x.left = tRight;
        	if (tRight != null) {
            	tRight.parent = x;
        	}

			//步骤c，t的父节点变为原先x的父节点
        	t.parent = parent;
        	if (parent == null) {
        		//若之前x为整棵树的根节点，则t接替它成为新的根节点mRoot
            	mRoot = t;
        	}
        	//t接替x成为x原先父节点的左孩子或右孩子
        	else if (parent.left == x) {
            	parent.left = t;
        	}
        	else if (parent.right == x) {
            	parent.right = t;
        	}
    	}
		```
		到这里，红黑树节点的左右旋转就分析完毕了，这两个操作在恢复红黑树平衡的过程中扮演了重要的作用，下面就来看看红黑树经过插入或删除操作后的调整过程。

	- **插入后调整**

		前面说过，红黑树的插入操作是在普通二叉查找树插入的基础上进行调整，所以将元素按键值大小插入到合适位置这一步跟前面介绍的二叉查找树的实现一致，在此基础上我们需要进行调整使红黑树继续满足其五条特性。
		
		首先，前三条特性不会因为一个节点的插入而被破坏，所以只需要关心后面两条即可。这里引用张拭心同学的文章中的一段话：
		
		“插入一个节点后要担心违反特征 4 和 5，数学里最常用的一个解题技巧就是把多个未知数化解成一个未知数。我们这里采用同样的技巧，把插入的节点直接染成红色，这样就不会影响特征 5，只要专心调整满足特征 4 就好了。这样比同时满足 4、5 要简单一些。染成红色后，我们只要关心父节点是否为红，如果是红的，就要把父节点进行变化，让父节点变成黑色，或者换一个黑色节点当父亲，这些操作的同时不能影响 不同路径上的黑色节点数一致的规则。”
		
		所以我们插入的节点均为红色，若插入后父节点为黑色则不用调整；若父节点为红色，则根据其叔叔节点的颜色有两种情况，下面我们分别来分析。（注：下图中，X是待添加节点，插入后X是作为L或是R的子树对于后续的调整操作是有影响的，但这两种情况下的操作是对称的，张拭心同学的文章介绍的是插入到L下的情况，为避免重复，本文选择另一种情况进行分析）。
		
		**第一种情况：父节点R为红色，叔叔节点L也为红色（显然爷爷节点P肯定是黑色）。**如下图左边所示，X和R形成了两个连续的红色节点，破坏了性质4，这时由于叔叔节点L也是红色，所以只需要把叔叔L和父亲R都染成黑色，同时将爷爷P染成红色就能使这一部分的红黑树重新恢复平衡（没有两个连续红色节点，同时每条路上黑色节点的数量也没有改变），即下图右边的情况。但此时P的染红会导致更上层的树结构被破坏，这时我们只需将P当做新插入的节点，再次向上进行相同的调整，以此循环直到父节点为黑色即可。
		
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/insert_1.png)
    
		**第二种情况：父节点R为红色，但叔叔节点和爷爷节点都为黑色。**如下图左边所示，这种情况下光靠染色已经不足以完成调整，因为若把R染成黑色或把L染成红色都会改变该条路径上黑色节点的个数，即会破坏性质5。此时该节点旋转登场了。
		
		考虑下图左边的情况，P-L路径两个黑色节点，P-R-X路径一个黑色节点，现在要求黑色节点个数不变，将红色的X接到一个黑色的父节点上，似乎那个红色的R是最大的阻碍，如果能把这个红色节点（注意，是红色节点，不是R节点）挪到左边那条路径去就能解决问题了。这时我们可以考虑对爷爷节点P进行左旋，这样一来爷爷节点就变成了红色的R，而黑色的P成了R的左孩子。然后我们再讲P和R的颜色交换，即R染成黑色，P染成红色，这样就得到了下图中右边的结构，可以看到左边R-P-L路径仍旧是两个黑色节点，右边R-X路径也依旧是一个黑色节点，同时也没有两个相连的红色节点，而且由于新的根节点R也为黑色，不需要再像情况1中那样循环向上进行调整，整棵红黑树已恢复平衡。
		
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/insert_2.png)
    
		另外，第二种情况下的调整策略也分为两种，上面是新插入节点X是其父节点R的右孩子时的情况，下面这种是X为R的左孩子的情况，如下图最左边所示，此时如果直接对P进行左旋，则X会成为P的右孩子，这样无法使树恢复平衡。所以需要多做一次旋转，先对节点R做一次右旋，得到下图中间所示结构，之后再对P进行左旋，并交换X和P的颜色即可。
		
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/insert_3.png)
    
		下面我们对照JDK1.8中HashMap新增的balanceInsertion方法源码过一下上文所述的过程：
		
		```
		//root是整棵树的根节点，x是新插入的节点，方法的返回值为插入操作后的根节点
		static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root, TreeNode<K,V> x) {
            //新插入的节点先染红
            x.red = true;
            //开始循环，xp为上图中的R(父亲)，xpp为P(爷爷)，
            //xppl为L(R为P的右孩子时，P的左孩子，即叔叔)，xppr是R为P的左孩子时P的右孩子，也是叔叔
            for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                if ((xp = x.parent) == null) {
                    x.red = false; //xp为空，即x为根节点，染成黑色，返回
                    return x;
                }
                else if (!xp.red || (xpp = xp.parent) == null) {
                    return root; //若xp是黑色，或者xp是根节点（也就等于xp是黑色），则无需调整，直接返回
                }
                
                //父节点xp为红色
                //如果x的父节点xp是爷爷节点xpp的左孩子
                if (xp == (xppl = xpp.left)) {
                		//如果叔叔节点也是红色
                    if ((xppr = xpp.right) != null && xppr.red) {
                        xppr.red = false; //叔叔染黑色
                        xp.red = false; //父亲染黑色
                        xpp.red = true; //爷爷染红色
                        x = xpp; //将x赋值为其爷爷节点xpp，继续向上层循环调整
                    }
                    //如果叔叔节点是黑色
                    else {
                        if (x == xp.right) {
                            root = rotateLeft(root, x = xp); //若x是其父节点的右孩子，则需要多做一次左旋
                            xpp = (xp = x.parent) == null ? null : xp.parent; //这时x和xp的层数会互换，x变为xp的父节点
                        }
                        if (xp != null) {
                        	//父节点xp和爷爷节点xpp互换颜色，并且对于爷爷节点xpp做右旋
                            xp.red = false; 
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateRight(root, xpp); //root有可能会改变，具体请参照rotateRightBST方法
                            }
                        }
                    }
                }
                //如果x的父节点xp是爷爷节点xpp的右孩子，这就是上文分析的那种情况，与上面if中的操作对称
                else {
                    if (xppl != null && xppl.red) {
                        xppl.red = false;
                        xp.red = false;
                        xpp.red = true;
                        x = xpp;
                    }
                    else {
                        if (x == xp.left) {
                            root = rotateRight(root, x = xp); //若x是其父节点的左孩子，则需要多做一次右旋
                            xpp = (xp = x.parent) == null ? null : xp.parent;
                        }
                        if (xp != null) {
                        	//父节点xp和爷爷节点xpp互换颜色，并且对于爷爷节点xpp做左旋
                            xp.red = false;
                            if (xpp != null) {
                                xpp.red = true;
                                root = rotateLeft(root, xpp);
                            }
                        }
                    }
                }
            }
        }
		```
		至此，红黑树插入节点后的调整操作就分析完毕了，整个过程通过对节点染色和对节点旋转这两个操作来恢复平衡，由于节点染色是一个非常快的操作，而旋转虽然较为复杂，但是对于插入调整来说至多只需要做两次旋转便可使整棵树恢复平衡，因此也不会对性能造成大的影响。
		
	- **删除后调整**

		和插入操作一样，红黑树的删除操作也是在二叉查找树删除操作的基础上进行必要的调整以恢复红黑树的平衡。同样的，红黑树的前三条性质不会被破坏，所以仅考虑第4，5条性质。
		
		显然，如果删除的是一个红色节点，则不会对树的平衡产生任何影响，即不需要调整。如果是一个黑色节点，则会导致其所在的子树路径上黑色节点的个数比另一棵子树路径的少，也即破坏了红黑树的第5条性质。所以我们要做的，**要么直接将另一条路径中的某一个黑色节点染红（不一定能这么做），要么通过节点旋转加染色来调整两边的节点。**说到底还是旋转和染色这两个手段。
    
		![](https://github.com/kfrozen/Red-Black-Tree_Note/blob/master/img/delete.png)
    
		考虑上图中的情况，最左边是删除前的部分红黑树，可以看到左右子树的每条路径上都是两个黑色节点，现在要删除节点X，删除后变为中间的不平衡结构，因为P的左子树少了一个黑色节点。现做如下调整：
		
		a. 首先将X的兄弟节点Y染成黑色，同时将X的父节点P染成红色；
		
		b. 接着对节点P做左旋转，然后由于L已经是叶子节点，所以染红并向上以P为对象继续循环调整，下一轮中由于P是红色，直接染成黑色并返回（这一步是根据源码得出，虽然在我们当前的这个例子中稍显累赘，但是毕竟我们举的是一个较简单的例子，实际的情况会更复杂），最终得到了上图最右边的结构，可以看到这时左右子树的每条路径上又变成了各自两个黑色节点，红黑树再次恢复了平衡。
		
		这便是删除后调整的一种情况，下面我们通过**HashMap中的balanceDeletion方法**分析删除调整可能遇到的所有情况。
		
		在开始看balanceDeletion方法的源码前，先看一下它被调用的地方，这里主要关注它的两个输入参数的含义：
		
		```
		final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab, boolean movable) {
		
			 ......
			
			 //p是待删除节点，replacement指的是删除p后需要来接替p位置的节点，当p为叶子节点时，p==replacement
			 if (replacement != p) {
			 	//若p不为叶子节点，则让继承者跟父节点相认
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }
            
            //若待删除的节点p时红色的，则树平衡未被破坏，无需进行调整。否则进行删除后调整，balanceDeletion方法就是在这里被调用的
            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);

			  //p为叶子节点，没有继承者，则将p从树中清除
            if (replacement == p) {  // detach
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
       }
		```
		可以看出，第一个输入参数是整棵红黑树的根节点，第二个输入参数是待删除节点或是其继承者，可以当做是上图中的X节点。搞清楚了输入参数，下面我们就开始分析丧心病狂的balanceDeletion方法。
		
		```
		//x就是上文中说到的replacement
		static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root, TreeNode<K,V> x) {
            for (TreeNode<K,V> xp, xpl, xpr;;)  {
                if (x == null || x == root)
                    return root; //x为空或x为根节点，返回
                else if ((xp = x.parent) == null) {
                    x.red = false; //x为根节点，染成黑色，返回
                    return x;
                }
                else if (x.red) {
                    x.red = false;
                    return root; //x为红色，则无需调整，返回
                }
                //x为其父节点的左孩子
                else if ((xpl = xp.left) == x) {
                    if ((xpr = xp.right) != null && xpr.red) { //有红色的兄弟节点xpr，则父亲节点xp必为黑色
                        xpr.red = false; //兄弟染成黑色
                        xp.red = true; //父亲染成红色
                        root = rotateLeft(root, xp); //对父节点xp做左旋转
                        xpr = (xp = x.parent) == null ? null : xp.right; //重新将xp指向原先x的父节点，xpr则指向xp新的右孩子
                    }
                    if (xpr == null)
                        x = xp; //若新的xpr为空，即上图中L节点为空，则向上继续调整，将x的父节点xp作为新的x继续循环
                    else {
                    	//此时的xpr就是上图中的节点L，sl和sr就是其左右孩子
                        TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                        if ((sr == null || !sr.red) &&
                            (sl == null || !sl.red)) {
                            xpr.red = true; //若sl和sr都为黑色或者不存在，即xpr没有红色孩子，则将xpr染红
                            x = xp; //本轮结束，继续向上循环，这就是上图中的情况，新的x即为红色的P节点，下一次循环时P会被染黑，然后循环结束
                        }
                        else {
                        	//否则的话，就需要进一步调整
                        	//现在的情况是被删除的X的路径需要一个新的黑色节点，即上图中P的左子树中，只能考虑从P的右子树搬运
                        	//所以最终要做的是对P做左旋，但L的左子树sl会在左旋后变为P的右子树，因此在左旋之前需要对sl和sr做处理
                            if (sr == null || !sr.red) { 
                                if (sl != null) //若左孩子为红，右孩子不存在或为黑
                                    sl.red = false; //左孩子染黑
                                xpr.red = true; //将xpr染红
                                root = rotateRight(root, xpr); //此时考虑上图，P和L均为红，于是需要右旋L将黑色的sl换过来
                                xpr = (xp = x.parent) == null ?
                                    null : xp.right;  //右旋后，xpr指向xp（上图P）的新右孩子，即上一步中的sl
                            }
                            if (xpr != null) {
                                xpr.red = (xp == null) ? false : xp.red; //xpr染成跟父节点一致的颜色，为后面父节点xp的左旋做准备
                                if ((sr = xpr.right) != null)
                                    sr.red = false; //xpr新的右孩子染黑，防止出现两个红色相连
                            }
                            if (xp != null) {
                                xp.red = false; //将xp染黑，并对其左旋，这样就能保证被删除的X所在的路径又多了一个黑色节点，从而达到恢复平衡的目的
                                root = rotateLeft(root, xp);
                            }
                            //到此调整已经完毕，故将x置为root，进入下一次循环后将直接退出
                            x = root;
                        }
                    }
                }
                //x为其父节点的右孩子，与上述过程对称
                else { // symmetric
                    if (xpl != null && xpl.red) {
                        xpl.red = false;
                        xp.red = true;
                        root = rotateRight(root, xp);
                        xpl = (xp = x.parent) == null ? null : xp.left;
                    }
                    if (xpl == null)
                        x = xp;
                    else {
                        TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                        if ((sl == null || !sl.red) &&
                            (sr == null || !sr.red)) {
                            xpl.red = true;
                            x = xp;
                        }
                        else {
                            if (sl == null || !sl.red) {
                                if (sr != null)
                                    sr.red = false;
                                xpl.red = true;
                                root = rotateLeft(root, xpl);
                                xpl = (xp = x.parent) == null ?
                                    null : xp.left;
                            }
                            if (xpl != null) {
                                xpl.red = (xp == null) ? false : xp.red;
                                if ((sl = xpl.left) != null)
                                    sl.red = false;
                            }
                            if (xp != null) {
                                xp.red = false;
                                root = rotateRight(root, xp);
                            }
                            x = root;
                        }
                    }
                }
            }
        }
		```
		以上就是HashMap中对于红黑树删除后调整的实现，逻辑很绕，但最终的目的就是让整棵树再次满足红黑树的5条特性。

- **最后**

	红黑树是一种特殊的二叉查找树，它的五条特性保证了即便在最差情况下其查找，插入和删除操作的复杂度仍为O(logN)。
	
	红黑树的查找算法和二叉查找树无异，插入和删除也是基于二叉查找树的做法，只是在其基础上需要进行调整以重新恢复树的平衡（主要是重新满足第4，5条特性）。
	
	由于其有序，快速的特点，红黑树在很多场景下都有被应用，比如Java中的TreeMap，以及Java 8中的HashMap。对于HashMap中红黑树的应用，后续会有文章单独进行分析。感谢阅读！
	

版权声明：原创不易，转载前请留言获得作者许可，转载后标明作者 Troy.Tang 与 原文链接。
	
	
	
	
