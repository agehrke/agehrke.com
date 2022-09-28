---
title: "Nested Set in Propel 1.3"
date: "2009-04-19"
categories: 
  - "symfony"
tags: 
  - "propel"
---

I'm currently working on a [Symfony](http://www.symfony-project.org/) project with a lot of [hierarchical data](http://dev.mysql.com/tech-resources/articles/hierarchical-data.html). For this I have taken advance of the new [Nested Set implementation in Propel 1.3](http://propel.phpdb.org/trac/wiki/Users/Documentation/1.3/Tree/NestedSet). Even though some initial research on Google showed a few people complaining about Propels implementation, I decided to give it a try. My schema.yml

```yaml
team:
  _attributes:
    treeMode: NestedSet
  id:
    company_id:
      type: integer
      treeScopeKey: true
    name:
      type: VARCHAR
      size: 128
      required: true
    lft:
      type: integer
      required: true
      default: 0
      nestedSetLeftKey: true
    rgt:
      type: integer
      required: true
      default: 0
      nestedSetRightKey: true
```

Currently the only problem I have had is when I need to reassign a new node as root node. My first thought was to make the new node root and add the old root node as child, but Propel doesn't like that and messes up the left-right values. What I have done is to insert the new node as child of the root node and then swap left-right values, like this:

```php
$node = new Team();
$root = TeamPeer::retrieveRoot($scopeId);
$node->insertAsFirstChildOf($root);

// Swap left-right values
$rootRight = $root->getRightValue();
$root->setLeftValue($node->getLeftValue());
$root->setRightValue($node->getRightValue());

// Make node root
$node->setLeftValue(1);
$node->setRightValue($rootRight);
```

### Display tree using RecursiveIteratorIterator

I retrieve the tree with the static peer method `retrieveTree()` and display it using the [RecursiveIteratorIterator](http://www.php.net/~helly/php/ext/spl/classRecursiveIteratorIterator.html). 

```php
$tree = TeamPeer::retrieveTree($scopeId);
$it = new RecursiveIteratorIterator($tree, RecursiveIteratorIterator::SELF\_FIRST); 
foreach ($it as $item) { echo $item; }
```

However I wanted to print the items as an indented "list" in a dropdown box. For this I extended the `RecursiveIteratorIterator` class and implemented its `beginChildren()` and `endChildren()` methods. 

```php
class MyIndentRecursiveIteratorIterator extends RecursiveIteratorIterator {
  private $indent = '';
  private $indentStr = '&nbsp;&nbsp;';
  
  public function getIndent() {
    return $this->indent;
  }
  
  public function beginChildren() {
    $this->indent = str\_repeat($this->indentStr, $this->getDepth());
  }
    
  public function endChildren() {
    $this->indent = str\_repeat($this->indentStr, $this->getDepth() - 1);
  }
}

$tree = TeamPeer::retrieveTree($scopeId);
$it = new MyIndentRecursiveIteratorIterator($tree, MyIndentRecursiveIteratorIterator::SELF\_FIRST);
foreach ($it as $item) { echo $it->getIndent() . $item; }
```

Which produces something like below which can be used in a dropdown or similar.

```
Root
  Child 1
    Child 1.1
    Child 1.2
  Child 2
```
