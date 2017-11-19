---
layout: post 
author: shalou
title:  "一致性hash的简单实现" 
category: linux
tag: [一致性hash]
---


一致性hash的两个核心

* ring
* 虚节点

<!-- more -->

```golang
package main

import (
    "crypto/md5"
    "fmt"
    "log"
    //"math/rand"
    "os"
    "sort"
    "strconv"
    "sync"
    //"time"
)

const VNodeNumPerNode int = 100 

type HashFunc func(string) string

type ConsistentHash struct {
    sync.RWMutex
    Nodes     []string
    vNodes    []string
    vNodesMap map[string]string
    hashFunc  HashFunc
}

func Md5Hash(key string) string {
    md5Ctx := md5.New()
    md5Ctx.Write([]byte(key))
    return string(md5Ctx.Sum(nil))
}

func NewConsistentHash(nodes []string, hashAlg HashFunc) (*ConsistentHash, error) {
    hash := &ConsistentHash{
        Nodes:     []string{},
        vNodes:    []string{},
        vNodesMap: make(map[string]string),
    } 
    
    if hashAlg != nil {
        hash.hashFunc = hashAlg
    } else {
        hash.hashFunc = Md5Hash
    }

    for _, node := range nodes {
        err := hash.AddNode(node)
        if err != nil {
            return nil, err
        }
    }

    return hash, nil
}

func nodeIndex(toAdd string, nodes []string) int {
    for index, node := range nodes {
        if toAdd == node {
            return index
        }
    }
    return -1
}

func (hash *ConsistentHash) AddNode(node string) error {
    hash.Lock()
    defer hash.Unlock()

    log.Println("add node ", node)

    if nodeIndex(node, hash.Nodes) != -1 {
        return fmt.Errorf("%s is already existed", node)
    }

    hash.Nodes = append(hash.Nodes, node)

    // add vnode
    for i := 0; i < VNodeNumPerNode; i++ {
        hashedPart := hash.hashFunc(node + ":part" + strconv.Itoa(i))
        hash.vNodes = append(hash.vNodes, hashedPart)
        hash.vNodesMap[hashedPart] = node
    }

    // sort the vnode
    sort.Strings(hash.vNodes)

    return nil
}

func (hash *ConsistentHash) DeleteNode(node string) error {
    hash.Lock()
    defer hash.Unlock()

    log.Printf("delete node %s", node)

    index := nodeIndex(node, hash.Nodes)
    if index == -1 {
        return fmt.Errorf("%s do not exist", node)
    }

    hash.Nodes = append(hash.Nodes[:index], hash.Nodes[index+1:]...)

    for i := 0; i < VNodeNumPerNode; i++ {
        hashedPart := hash.hashFunc(node + ":part" + strconv.Itoa(i))
        len := len(hash.vNodes)

        // delete the vNode
        vIndex := sort.Search(len, func(i int) bool { return hash.vNodes[i] >= hashedPart })
        if vIndex == len {
            log.Fatalf("index %d, %x does't exist", i, hashedPart)
        } else {
            hash.vNodes = append(hash.vNodes[:vIndex], hash.vNodes[vIndex+1:]...)
        }
        // no need sort

        delete(hash.vNodesMap, hashedPart)
    }

    return nil
}

func (hash *ConsistentHash) GetServer(key string) string {
    hash.RLock()
    defer hash.RUnlock()

    if len(hash.Nodes) == 0 {
        return ""
    }

    hashedKey := hash.hashFunc(key)

    // Get the first vNode whose value greater than hashedKey
    vIndex := sort.Search(len(hash.vNodes), func(i int) bool { return hash.vNodes[i] >= hashedKey })

    // return the first node
    if vIndex == len(hash.vNodes) {
        vNode := hash.vNodes[0]
        node, ok := hash.vNodesMap[vNode]
        if !ok {
            log.Fatal("interal error")
        }
        return node
    } else {
        vNode := hash.vNodes[vIndex]
        node, ok := hash.vNodesMap[vNode]
        if !ok {
            log.Fatal("interal error")
        }
        return node
    }
}

func (hash *ConsistentHash) GetNodes() []string {
    return hash.Nodes
}

func (hash *ConsistentHash) GetVNodes() []string {
    return hash.vNodes
}

func main() {
    servers := []string{"192.168.101.1:8080", "192.168.101.2:8080", "192.168.101.3:8080"}
    hash, err := NewConsistentHash(servers, nil)
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }

    if err := hash.AddNode("192.168.101.4:8080"); err != nil {
        log.Println(err)
    }

    if err := hash.DeleteNode("192.168.101.1:8080"); err != nil {
        log.Println(err)
    }

    //var inputKeys [10000]int
    var counter = make(map[string]int)

    for _, node := range hash.GetNodes() {
        counter[node] = 0
    }

    //r := rand.New(rand.NewSource(time.Now().UnixNano()))

    for i := 0; i < 1000000; i++ {
        server := hash.GetServer(strconv.Itoa(i))
        counter[server]++
    }

    for k, v := range counter {
        fmt.Println(k, v)
    }

}
```
