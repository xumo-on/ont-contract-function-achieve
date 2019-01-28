### 1.合约框架

​	以hello world合约为例：

```python
from ontology.interop.System.Runtime import Notify

def Main(operation, args):
    if operation == 'Hello':
        msg = args[0]
        return Hello(msg)

    return False


def Hello(msg):
    Notify(msg)
    return True
```

- 目前智能合约都建议使用python来编写。
- `from ontology.interop.System.Runtime import Notify`是引用，引用的`Notify`为[ontology-python-compiler](https://github.com/ontio/ontology-python-compiler)编译器中的`API`。
- `def Main(operation, args):`是主函数，传入两个参数，`operation`为调用的成员函数名，`args`为需要传入的其他参数。
- `def Hello(msg):`是成员函数，实现所需要的功能。
- 实现的功能是把传入的参数打印出来，假如输入`string`格式的`Hello World`，就会`Notify`出结果`48656c6c6f20576f726c64`，结果是`hex string`，通过工具转成`string`。

### 2.主函数

​	主函数的格式：

```python
def Main(operation, args):
    if operation == 'Hello':
        msg = args[0]
        return Hello(msg)

    return False
```

- `operation`为想要调用的成员函数名，为`string`格式，如果想要调用Hello函数，就需要输入`operation`为`Hello`。
- `args`为传入的其他参数，为`list`格式，`args[0]`是除`operation`外传入的第一个参数，假如传入的为`string`格式的`Hello`，则`args[0] = "Hello"`。
- 只要在主函数中写好`if operation == 'Hello':`格式的接口，就能够调用合约中的其他成员函数，没有写接口的不能调用(可以用于只在函数内部调用的功能)，`operation`的值必须与调用的函数名相同。

### 3.成员函数

​	成员函数的格式：

```python
def Hello(msg):
    Notify(msg)
    return True
```

- 如果直接`return msg`，就可以使用预执行来得到返回的值。

### 4.与合约交互的方法

- `pythonSdk`

  首先引用`sdk`并配置好rpc，钱包与合约地址：

  ```python
  from ontology.ont_sdk import OntologySdk # 引用sdk
  
  sdk = OntologySdk()
  rpc_address = 'http://127.0.0.1:20336'
  sdk.rpc.set_address(rpc_address)
  
  wallet_path = "D:\\xx\\wallet.dat"
  sdk.wallet_manager.open_wallet(wallet_path)
  admin_addr = "ARDskfQuvTa7TvbCVNX8MP5zjg7SUdDgSa" # base58 address
  admin_pwd = "123123"
  adminAcct = sdk.wallet_manager.get_account(admin_addr, admin_pwd)
  
  ContractAddress = "6bc94156ad2057cab6b159217d4e444fd2e17005"
  contract_address_str = lContractAddress
  contract_address = bytearray.fromhex(contract_address_str)
  contract_address.reverse() # 传入的哈希必须要反序
  ```

  构造params：

  ```python
  
  ```

  构造交易调用函数：

  ```python
  tx = NeoVm.make_invoke_transaction(bytearray(contract_address), bytearray(params), b'', 20000, 500)
  sdk.sign_transaction(tx, payerAcct)
  res = sdk.rpc.send_raw_transaction_pre_exec(tx)
  ```

- `golang`

  获取合约地址以及签名人地址：

  ```go
  code = "123" // 合约编译出的AVM字节码
  codeAddress, _ = utils.GetContractAddress(code)
  
  signer, err := ctx.GetDefaultAccount()
  if err != nil {
  	ctx.LogError("TestDomainSmartContract GetDefaultAccount error:%s", err)
  	return false
  }
  ```

  部署合约(也可以在samartX上完成)：

  ```go
  _, err = ctx.Ont.NeoVM.DeployNeoVMSmartContract(ctx.GetGasPrice(), ctx.GetGasLimit(),
  	signer,
  	true,
  	code,
  	"TestDomainSmartContract",
  	"1.0",
  	"",
  	"",
  	"",
  )
  if err != nil {
  	ctx.LogError("TestDomainSmartContract DeploySmartContract error: %s", err)
  	return false
  }
  
  // 等待出块
  _, err = ctx.Ont.WaitForGenerateBlock(30*time.Second, 1)
  if err != nil {
  	ctx.LogError("TestDomainSmartContract WaitForGenerateBlock error: %s", err)
  	return false
  }
  ```

  预执行：

  ```go
  value, err := ctx.Ont.NeoVM.PreExecInvokeNeoVMContract(codeAddress, []interface{}{"getHeight", []interface{}{}})
  if err != nil {
  	ctx.LogError("TestDomainSmartContract PreExecInvokeSmartContract error: %s", err)
  	return false
  }
  bValue, err := value.Result.ToByteArray()
  Svalue := hex.EncodeToString(bValue)
  ```

### 5.内外接收数据的方法

合约内：

​	直接`return msg`，然后在另一个函数中调用这个函数，就可以得到传出的数据，如：`msg = Hello(msg)`。

合约外：

- `pythonSdk`想要获取合约传出的参数有三种方法

- 第一种是通过预执行，获取某个函数`return`的数据：

  ```python
  def test_invokeRead(self, payerAcct, param_list):
      params = BuildParams.create_code_params_script(param_list)
      tx = NeoVm.make_invoke_transaction(bytearray(contract_address), bytearray(params), b'', 20000, 500)
      sdk.sign_transaction(tx, payerAcct)
      res = sdk.rpc.send_raw_transaction_pre_exec(tx)
      return res
  ```

  步骤：

  1. 用`param_list`构造传入的`params`，类似`golang`中的`interface`
  2. 构造交易`transaction`
  3. 签名交易`sdk.sign_transaction(tx, payerAcct)`
  4. 预执行交易并返回结果`sdk.rpc.send_raw_transaction_pre_exec(tx)`

- 第二种是直接执行函数，通过返回的`hash`来获取函数中`Notify`出来的值：

  ```python
  def test_invoke(self, payerAcct, param_list):
      params = BuildParams.create_code_params_script(param_list)
          
      # 执行函数时构造params所必须的神秘步骤
      params.append(0x67)
      for i in contract_address:
          params.append(i)
              
      gaslimit = 20000
      unix_time_now = int(time.time())
      tx = Transaction(0, 0xd1, unix_time_now, 500, gaslimit, payerAcct.get_address().to_array(), params, bytearray(),
                           [], bytearray())
      sdk.sign_transaction(tx, payerAcct)
      hash = sdk.rpc.send_raw_transaction(tx)
      res = sdk.rpc.get_smart_contract_event_by_tx_hash(hash)
      events = res["Notify"]
          
      # string格式的notify
      notify = (bytearray.fromhex(event["States"][0])).decode('utf-8')
          
      # int格式的notify
      notify = event["States"][0]
          
      # address格式的notify
      account = Address(binascii.a2b_hex(effectiveEscapeAcctPointOddsProfit[0]))
      notify = account.b58encode()
          
      print(notify)
      return True
  ```

- 第三种是在合约内部把数`Put`到`storage`，然后再在`sdk`中调用：

  ```
  sdk.rpc.get_storage(contract_address, key)
  ```

- `golangSdk`获取参数也有三种方法

- 第一种预执行

  ```go
  res, err := ctx.Ont.NeoVM.PreExecInvokeNeoVMContract(
  	codeAddress,
      []interface{}{"hello", []interface{}{[]byte("hello")}})
  
  if err != nil {
  	ctx.LogError("error: %s", err)
  	return false
  }
  
  result := res.Result.ToByteArray() # .ToInteger  .ToString  .ToBool  .ToArray
  
  //由于从合约中取出的数据全部都是反序的，需要反序回来
  Svalue := hex.EncodeToString(result)
  
  count := strings.Count(Svalue, "") - 1
  s := []string{}
  for i := count; i > 0; i -= 2 {
  	s = append(s, Svalue[i-2:i])
  }
  s1 := strings.Join(s, "")
  
  result1, err := strconv.ParseUint(s1, 16, 32)
  ```

  `golang`中需要传入`interface`，具体的格式如上，其中第一个`interface`中的第一个参数`“Hello”`就是`operation`即想要调用的成员函数名，第二个`interface`中是可以传入的参数即`args`，如果没有传入就为空，但是一定要有两个`interface`嵌套。

- 第二种直接执行

  ```go
  txHash, err := ctx.Ont.NeoVM.InvokeNeoVMContract(ctx.GetGasPrice(), ctx.GetGasLimit(),
  	signer,
  	codeAddress,
  	[]interface{}{"Hello", []interface{}{[]byte("Hello")}})
  if err != nil {
  	ctx.LogError("TestDomainSmartContract InvokeNeoVMSmartContract error: %s", err)
  	return false
  }
  
  //等待出块完成交易
  _, err = ctx.Ont.WaitForGenerateBlock(30*time.Second, 1)
  if err != nil {
  	ctx.LogError("TestDomainSmartContract WaitForGenerateBlock error: %s", err)
  	return false
  }
  
  //由交易哈希获取交易的events
  events, err := ctx.Ont.GetSmartContractEvent(txHash.ToHexString())
  if err != nil {
  	ctx.LogError("TestInvokeSmartContract GetSmartContractEvent error:%s", err)
  	return false
  }
  if events.State == 0 {
  	ctx.LogError("TestInvokeSmartContract failed invoked exec state return 0")
  	return false
  }
  
  //由events获取notify
  notify := events.Notify[0]
  ```

- 第三种通过`sdk`调用`storage`

  ```go
  hash, err := ctx.Ont.GetStorage(codeAddress.ToHexString(), []byte("key"))
  if err != nil {
  	ctx.LogError("TestDomainSmartContract GetStorageItem key:hello error: %s", err)
  	return false
  }
  value := hex.EncodeToString(hash)
  ```

### 6.storage的使用与技巧

`python api`中的`storage api`实现了合约向`storage`的存储及调用。

- 引用`storage api`，定义`key`

  ```python
  from ontology.interop.System.Storage import GetContext, Get, Put, Delete
  
  INITIALIZED = "Init"
  ```

- 根据`context`从`storage`中调用数据`get`：

  ```python
  inited = Get(GetContext(), INITIALIZED)
  ```

- 根据`context`向`storage`中存储数据`put`：

  ```python
  Put(GetContext(), INITIALIZED, 'Y')
  ```

- 建议使用`concatKey`来创建新的`key`值，便于记忆与存储调用：

  ```python
  def concatKey(str1,str2):
      """
      connect str1 and str2 together as a key
      :param str1: string1
      :param str2:  string2
      :return: string1_string2
      """
      return concat(concat(str1, '_'), str2)
  ```

- 存储新的`key-value`值：

  ```python
  # ROUND_PREFIX = "G01", STATUS_OFF = "END"
  Put(GetContext(), concatKey(concatKey(ROUND_PREFIX, 0), ROUND_STATUS), STATUS_OFF)
  ```

  此处即为将END字符串，存储到键值G01_0对应的存储中。

### 7.admin操作

- 在一些只允许admin操作的函数中，建议在合约中使用`RequireWitness(Admin)`来校验交易的发起者是否为已定义好的admin，该函数同样可以用来验签。

  ```python
  Admin = Base58ToAddress("XXXX")
  
  RequireWitness(Admin)
  
  def RequireWitness(witness):
      """
  	Checks the transaction sender is equal to the witness. If not
  	satisfying, revert the transaction.
  	:param witness: required transaction sender
  	:return: True if transaction sender or revert the transaction.
  	"""
      Require(CheckWitness(witness))
      return True
  ```

- 迁移合约，更新合约，通过`api migrate`来实现，在原合约中建议再加上转账操作，避免迁移合约后，旧合约被删除，所有接口无法调用，导致旧合约中的ong或ont无法转账出来。

  ```python
  def migrateContract(code, needStorage, name, version, author, email, description, newReversedContractHash):
      RequireWitness(Admin)
      res = _transferONGFromContact(newReversedContractHash, getTotalONG())
      Require(res)
      if res == True:
          res = Migrate(code, needStorage, name, version, author, email, description)
          Require(res)
          Notify(["Migrate Contract successfully", Admin, GetTime()])
          return True
      else:
          Notify(["MigrateContractError", "transfer ONG to new contract error"])
          return False
  ```

### 8.SafeCheck和SafeMath

- SafeCheck

  ```python
  """
  https://github.com/ONT-Avocados/python-template/blob/master/libs/Utils.py
  """
  def Revert():
      """
      Revert the transaction. The opcodes of this function is `09f7f6f5f4f3f2f1f000f0`,
      but it will be changed to `ffffffffffffffffffffff` since opcode THROW doesn't
      work, so, revert by calling unused opcode.
      """
      raise Exception(0xF1F1F2F2F3F3F4F4)
  
  
  """
  https://github.com/ONT-Avocados/python-template/blob/master/libs/SafeCheck.py
  """
  def Require(condition):
      """
  	If condition is not satisfied, return false
  	:param condition: required condition
  	:return: True or false
  	"""
      if not condition:
          Revert()
      return True
  
  def RequireScriptHash(key):
      """
      Checks the bytearray parameter is script hash or not. Script Hash
      length should be equal to 20.
      :param key: bytearray parameter to check script hash format.
      :return: True if script hash or revert the transaction.
      """
      Require(len(key) == 20)
      return True
  
  def RequireWitness(witness):
      """
  	Checks the transaction sender is equal to the witness. If not
  	satisfying, revert the transaction.
  	:param witness: required transaction sender
  	:return: True if transaction sender or revert the transaction.
  	"""
      Require(CheckWitness(witness))
      return True
  ```

- SafeMath

  ```python
  """
  https://github.com/ONT-Avocados/python-template/blob/master/libs/SafeMath.py
  """
  
  def Add(a, b):
      """
      Adds two numbers, throws on overflow.
      """
      c = a + b
      Require(c >= a)
      return c
  
  def Sub(a, b):
      """
      Substracts two numbers, throws on overflow (i.e. if subtrahend is greater than minuend).
      :param a: operand a
      :param b: operand b
      :return: a - b if a - b > 0 or revert the transaction.
      """
      Require(a>=b)
      return a-b
  
  def ASub(a, b):
      if a > b:
          return a - b
      if a < b:
          return b - a
      else:
          return 0
  
  def Mul(a, b):
      """
      Multiplies two numbers, throws on overflow.
      :param a: operand a
      :param b: operand b
      :return: a - b if a - b > 0 or revert the transaction.
      """
      if a == 0:
          return 0
      c = a * b
      Require(c / a == b)
      return c
  
  def Div(a, b):
      """
      Integer division of two numbers, truncating the quotient.
      """
      Require(b > 0)
      c = a / b
      return c
  
  def Pwr(a, b):
      """
      a to the power of b
      :param a the base
      :param b the power value
      :return a^b
      """
      c = 0
      if a == 0:
          c = 0
      elif b == 0:
          c = 1
      else:
          i = 0
          c = 1
          while i < b:
              c = Mul(c, a)
              i = i + 1
      return c
  
  def Sqrt(a):
      """
      Return sqrt of a
      :param a:
      :return: sqrt(a)
      """
      c = Div(Add(a, 1), 2)
      b = a
      while(c < b):
          b = c
          c = Div(Add(Div(a, c), c), 2)
      return c
  ```

### 9.引入token

- token合约的部署

  参考[lucky合约](https://github.com/skyinglyh1/temporary-Moon/blob/master/luckyMoon2/Lucky.py)(oep4)

- token合约的调用

  ```python
  TOKEN_CONTRACT_HASH_KEY = "TokenContract"
  
  def setTokenContractHash(reversedTokenContractHash):
      RequireWitness(Admin)
      Put(GetContext(), TOKEN_CONTRACT_HASH_KEY, reversedTokenContractHash)
      Notify(["setTokenContractHash", reversedTokenContractHash])
      return True
  
  params = [ContractAddress, account, acctTokenBalanceToBeAdd]
  revesedContractAddress = Get(GetContext(), TOKEN_CONTRACT_HASH_KEY)
  res = DynamicAppCall(revesedContractAddress, "transfer", params)
  ```


### 10.合约转账与充值提现操作

- 转账

  ```python
  def _transferONG(fromAcct, toAcct, amount):
      """
      transfer ONG
      :param fromacct:
      :param toacct:
      :param amount:
      :return:
      """
      RequireWitness(fromAcct)
      param = state(fromAcct, toAcct, amount)
      res = Invoke(0, ONGAddress, 'transfer', [param])
      if res and res == b'\x01':
          return True
      else:
          return False
  ```

- 合约向账户转账

  ```python
  ONGAddress = bytearray(b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02')
  
  def _transferONGFromContact(toAcct, amount):
      param = state(ContractAddress, toAcct, amount)
      res = Invoke(0, ONGAddress, 'transfer', [param])
      if res and res == b'\x01':
          return True
      else:
          return False
  ```

- admin向合约充值

  ```python
  def adminInvest(ongAmount):
      RequireWitness(Admin)
      Require(_transferONG(Admin, ContractAddress, ongAmount))
      Put(GetContext(), TOTAL_ONG_FOR_ADMIN, Add(getTotalOngForAdmin(), ongAmount))
      Notify(["adminInvest", ongAmount])
      return True
  ```

- admin提现lucky token

  ```python
  def adminWithdrawLucky(toAcct, luckyAmount):
      RequireWitness(Admin)
      revesedContractAddress = getLuckyContractHash()
      params = [ContractAddress]
      totalLuckyAmount = DynamicAppCall(revesedContractAddress, "balanceOf", params)
      if luckyAmount <= totalLuckyAmount:
          params = [ContractAddress, toAcct, luckyAmount]
          res = DynamicAppCall(revesedContractAddress, "transfer", params)
          Require(res)
      Notify(["adminWithdrawLucky", toAcct, luckyAmount])
      return True
  ```

- admin提现ong

  ```python
  def adminWithdraw(toAcct, ongAmount):
      RequireWitness(Admin)
      roundId = getCurrentRound()
  
      Require(getRoundGameStatus(roundId) == 2)
      totalOngForAdmin = getTotalOngForAdmin()
      Require(ongAmount <= totalOngForAdmin)
      Put(GetContext(), TOTAL_ONG_FOR_ADMIN, Sub(getTotalOngForAdmin(), ongAmount))
  
      Require(_transferONGFromContact(toAcct, ongAmount))
      Notify(["adminWithdraw", toAcct, ongAmount])
      return True
  
  def getTotalOngForAdmin():
      return Get(GetContext(), TOTAL_ONG_FOR_ADMIN)
  ```

### 11.添加邀请人与邀请人分红

- 设置分红参数

  ```python
  def setReferralBonusPercentage(referralBonus):
      RequireWitness(Admin)
      Require(referralBonus >= 0)
      Require(referralBonus <= 100)
      Put(GetContext(), REFERRAL_BONUS_PERCENTAGE_KEY, referralBonus)
      Notify(["setReferralBonus", referralBonus])
      return True
  
  def setLuckyToOngRate(ong, lucky):
      """
      Say, if the player puts 1 ONG and he can get 0.5 Lucky, then ong = 2, lucky =1, please neglect the decimals
      :param ong:
      :param lucky:
      :return:
      """
      RequireWitness(Admin)
      Put(GetContext(), LUCKY_TO_ONG_RATE_KEY, Div(Mul(lucky, Magnitude), ong))
      Notify(["setRate", ong, lucky])
      return True
  
  setReferralBonusPercentage(10)
  setLuckyToOngRate(1, 2)
  ```

- 查看分红参数

  ```python
  def getReferralBonusPercentage():
      return Get(GetContext(), REFERRAL_BONUS_PERCENTAGE_KEY)
  
  def getLuckyToOngRate():
      """
      Div(Mul(Mul(lucky, LuckyMagnitude), Magnitude), Mul(ong, ONGMagnitude))
       lucky
      ------- * Magnitude
       ong
      :return:
      """
      return Get(GetContext(), LUCKY_TO_ONG_RATE_KEY)
  ```

- 添加邀请人

  ```python
  def addReferral(toBeReferred, referral):
      RequireScriptHash(toBeReferred)
      RequireScriptHash(referral)
      if CheckWitness(Admin) or CheckWitness(toBeReferred):
          if not getReferral(toBeReferred):
              Put(GetContext(), concatKey(PLAYER_REFERRAL_KEY, toBeReferred), referral)
              Notify(["addReferral", toBeReferred, referral])
              return True
      return False
  ```

- 查看邀请人

  ```python
  def getReferral(toBeReferred):
      return Get(GetContext(), concatKey(PLAYER_REFERRAL_KEY, toBeReferred))
  ```
