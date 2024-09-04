##rock-paper-scissor

这个题就是个石头剪刀布游戏，关键是这个函数，生成一个随机的石头剪刀布，按理说我们就选石头然后爆破也能成功
```solidity
function randomShape() internal view returns (Hand) {
        return Hand(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number - 1)))) % 3);
    }
```

当然最优解还是计算对应的哈希，由于顺序为石头布剪刀，所以我们只要哈希值大1就可以了

```solidity
    uint256(keccak256(abi.encodePacked(msg.sender, <HASH>))) % 3;//辅助计算传递的yours函数
```
