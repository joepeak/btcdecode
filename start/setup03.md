#   如何接入比特币网络以及原理分析

以下内容为系统启动过程中，每一步骤的详细分析。


##  第3步，参数到内部标志的转换处理（`src/bitcoind.cpp`）

`AppInitParameterInteraction` 函数后半部分。

1.  处理 `debug`、`debugexclude`、`debugnet` 等参数。

    如果指定了 `-debug`，则解析每个类别是否是支持的类别。如果不支持，则输入警告消息。如果需要同时指定多个类别，可分开指定，比如要调试网络与RPC 相关的信息，则配置如下：`-debug=net -debug=rpc`。

        if (gArgs.IsArgSet("-debug")) {
            const std::vector<std::string> categories = gArgs.GetArgs("-debug");

            if (std::none_of(categories.begin(), categories.end(),
                [](std::string cat){return cat == "0" || cat == "none";})) {
                for (const auto& cat : categories) {
                    if (!g_logger->EnableCategory(cat)) {
                        InitWarning(strprintf(_("Unsupported logging category %s=%s."), "-debug", cat));
                    }
                }
            }
        }

2.  如果指定了 `-socks` 参数，则提示使用 SOCKS5

        if (gArgs.IsArgSet("-socks"))
            return InitError(_("Unsupported argument -socks found. Setting SOCKS version isn't possible anymore, only SOCKS5 proxies are supported."));

3.  如果指定了 `-tor` 参数，则提示使用 `-onion`。

        if (gArgs.GetBoolArg("-tor", false))
            return InitError(_("Unsupported argument -tor found, use -onion."));

4.  如果指定了 `-benchmark` 参数，则提示使用 `-debug=bench`。

        if (gArgs.GetBoolArg("-benchmark", false))
            InitWarning(_("Unsupported argument -benchmark ignored, use -debug=bench."));

5.  如果指定了 `-whitelistalwaysrelay` 参数，则提示使用 `-whitelistrelay`，或`-whitelistforcerelay`。

        if (gArgs.GetBoolArg("-whitelistalwaysrelay", false))
            InitWarning(_("Unsupported argument -whitelistalwaysrelay ignored, use -whitelistrelay and/or -whitelistforcerelay."));

6.  如果指定了 `-blockminsize` 参数，则提示使用 `-blockminsize`。

        if (gArgs.IsArgSet("-blockminsize"))
            InitWarning("Unsupported argument -blockminsize ignored.");

7.  根据是否指定 `-checkmempool` 参数，确定是否进行合理性检查。

    在不指定这个参数的情况下，当运行主网络和测试网络时，不进行交易池合理性检查，当运行回归测试网络时，进行合理性检查。代码如下：

        // Checkmempool and checkblockindex default to true in regtest mode

        // 当运行主网络和测试网络时，DefaultConsistencyChecks 函数返回假，导致变量 ratio 为1， 为0，从而不进行交易池设置；当运行回归测试网络时，函数返回真，从而变量 ratio 为1，从而进行交易池设置
        int ratio = std::min<int>(std::max<int>(gArgs.GetArg("-checkmempool", chainparams.DefaultConsistencyChecks() ? 1 : 0), 0), 1000000);
        if (ratio != 0) {
            mempool.setSanityCheck(1.0 / ratio);
        }

        fCheckBlockIndex = gArgs.GetBoolArg("-checkblockindex", chainparams.DefaultConsistencyChecks());

8.  设置检查点默认打开

        fCheckpointsEnabled = gArgs.GetBoolArg("-checkpoints", DEFAULT_CHECKPOINTS_ENABLED);

9.  处理 `assumevalid` 参数。

    这个参数的意思是假设有效，即如果如果这个块在链中，则假定它和它的祖先是有效的，并且可能跳过它们的脚本验证。否则会验证所有的块。

        hashAssumeValid = uint256S(gArgs.GetArg("-assumevalid", chainparams.GetConsensus().defaultAssumeValid.GetHex()));
        if (!hashAssumeValid.IsNull())
            LogPrintf("Assuming ancestors of block %s have valid signatures.\n", hashAssumeValid.GetHex());
        else
            LogPrintf("Validating signatures for all blocks.\n");

10.  根据是否指定 `minimumchainwork` 参数，确定使用默认的最小工作量还是使用用户指定的最小工作量。

            if (gArgs.IsArgSet("-minimumchainwork")) {
                const std::string minChainWorkStr = gArgs.GetArg("-minimumchainwork", "");
                if (!IsHexNumber(minChainWorkStr)) {
                    return InitError(strprintf("Invalid non-hex (%s) minimum chain work value specified", minChainWorkStr));
                }
                nMinimumChainWork = UintToArith256(uint256S(minChainWorkStr));
            } else {
                nMinimumChainWork = UintToArith256(chainparams.GetConsensus().nMinimumChainWork);
            }

11. 计算内存池/交易池限制，包括处理 `maxmempool`、`limitdescendantsize` 参数。

    前者表示最大内存池，后者表示最小内存池，如果最大值小于最小值，则抛出初始化错误。

        int64_t nMempoolSizeMax = gArgs.GetArg("-maxmempool", DEFAULT_MAX_MEMPOOL_SIZE) * 1000000;
        int64_t nMempoolSizeMin = gArgs.GetArg("-limitdescendantsize", DEFAULT_DESCENDANT_SIZE_LIMIT) * 1000 * 40;
        if (nMempoolSizeMax < 0 || nMempoolSizeMax < nMempoolSizeMin)
            return InitError(strprintf(_("-maxmempool must be at least %d MB"), std::ceil(nMempoolSizeMin / 1000000.0)));

12. 如果指定了 `incrementalrelayfee`，则进行相关处理。

    `incrementalrelayfee` 定义中继的成本费率，应用于交易池限制和 BIP 125 替换。代码如下：

        if (gArgs.IsArgSet("-incrementalrelayfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-incrementalrelayfee", ""), n))
                return InitError(AmountErrMsg("incrementalrelayfee", gArgs.GetArg("-incrementalrelayfee", "")));
            incrementalRelayFee = CFeeRate(n);
        }
    
13. 处理 `-par` 参数，指定脚本签名的线程数量。

    代码如下：

        if (nScriptCheckThreads <= 0)
            nScriptCheckThreads += GetNumCores();
        if (nScriptCheckThreads <= 1)
            nScriptCheckThreads = 0;
        else if (nScriptCheckThreads > MAX_SCRIPTCHECK_THREADS)
            nScriptCheckThreads = MAX_SCRIPTCHECK_THREADS;

14. 处理区块修剪参数 `-prune`。

    代码如下：

        int64_t nPruneArg = gArgs.GetArg("-prune", 0);
        if (nPruneArg < 0) {
            return InitError(_("Prune cannot be configured with a negative value."));
        
        nPruneTarget = (uint64_t) nPruneArg * 1024 * 1024;
        if (nPruneArg == 1) {  // manual pruning: -prune=1
            LogPrintf("Block pruning enabled.  Use RPC call pruneblockchain(height) to manually prune block and undo files.\n");
            nPruneTarget = std::numeric_limits<uint64_t>::max();
            fPruneMode = true;
        } else if (nPruneTarget) {
            if (nPruneTarget < MIN_DISK_SPACE_FOR_BLOCK_FILES) {
                return InitError(strprintf(_("Prune configured below the minimum of %d MiB.  Please use a higher number."), MIN_DISK_SPACE_FOR_BLOCK_FILES / 1024 / 1024));
            }
            LogPrintf("Prune configured to target %uMiB on disk for block and undo files.\n", nPruneTarget / 1024 / 1024);
            fPruneMode = true;
        }

15. 处理连接超时时间 `-timeout`。

    代码如下：

        nConnectTimeout = gArgs.GetArg("-timeout", DEFAULT_CONNECT_TIMEOUT);
        if (nConnectTimeout <= 0)
            nConnectTimeout = DEFAULT_CONNECT_TIMEOUT;

16. 处理 `-minrelaytxfee` 参数。

    对于中继、挖矿和交易创建，小于此的费用被认为是零费用。解析并计算最小中继交易费用。代码如下：

        if (gArgs.IsArgSet("-minrelaytxfee")) {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-minrelaytxfee", ""), n)) {
                return InitError(AmountErrMsg("minrelaytxfee", gArgs.GetArg("-minrelaytxfee", "")));
            }
            ::minRelayTxFee = CFeeRate(n);
        } else if (incrementalRelayFee > ::minRelayTxFee) {
            ::minRelayTxFee = incrementalRelayFee;
            LogPrintf("Increasing minrelaytxfee to %s to match incrementalrelayfee\n",::minRelayTxFee.ToString());
        }

17. 处理 `-blockmintxfee` 参数。

    设置要在块创建中包含的事务满足的最低费率，即低于这个费率，交易将不进行打包。代码如下：

        if (gArgs.IsArgSet("-blockmintxfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-blockmintxfee", ""), n))
                return InitError(AmountErrMsg("blockmintxfee", gArgs.GetArg("-blockmintxfee", "")));
        }

18. 处理 `-dustrelayfee` 参数。

    具体代码如下：

        if (gArgs.IsArgSet("-dustrelayfee"))
        {
            CAmount n = 0;
            if (!ParseMoney(gArgs.GetArg("-dustrelayfee", ""), n))
                return InitError(AmountErrMsg("dustrelayfee", gArgs.GetArg("-dustrelayfee", "")));
            dustRelayFee = CFeeRate(n);
        }

19. 处理 `-acceptnonstdtxn` 参数。

    这个参数代表中继和挖掘非标准交易。具体代码如下：

        fRequireStandard = !gArgs.GetBoolArg("-acceptnonstdtxn", !chainparams.RequireStandard());
        if (chainparams.RequireStandard() && !fRequireStandard)
            return InitError(strprintf("acceptnonstdtxn is not currently supported for %s chain", chainparams.NetworkIDString()));

20. 处理 `-bytespersigop` 参数。

    计算中继和挖掘中交易的 sigop 的等效字节数。

        nBytesPerSigOp = gArgs.GetArg("-bytespersigop", nBytesPerSigOp);

21. 调用钱包初始接口对象的 `ParameterInteraction` 方法，初始钱包相关的参数。本方法在 `wallet/init.cpp` 文件中。

    调用代码如下：

        if (!g_wallet_init_interface.ParameterInteraction()) return false;

    代码内部具体处理如下：

    -   检查是否禁用钱包 `-disablewallet`。如果禁用，就不会加载钱包，并且会禁用钱包 RPC，这种情况下忽略 `-wallet`。

            if (gArgs.GetBoolArg("-disablewallet", DEFAULT_DISABLE_WALLET)) {
                for (const std::string& wallet : gArgs.GetArgs("-wallet")) {
                    LogPrintf("%s: parameter interaction: -disablewallet -> ignoring -wallet=%s\n", __func__, wallet);
                }

                return true;
            }

    -   确定指定的是单一钱包还是多钱包。

            gArgs.SoftSetArg("-wallet", "");
            const bool is_multiwallet = gArgs.GetArgs("-wallet").size() > 1;

    -   处理 `-blocksonly`、`-walletbroadcast` 参数。

    -   处理 `-salvagewallet`、`-rescan` 参数。

        如果指定了在启动时试图从损坏的钱包中恢复私钥，即 `-salvagewallet` 参数，那么不能使用多个钱包。

            if (gArgs.GetBoolArg("-salvagewallet", false)) {
                if (is_multiwallet) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-salvagewallet"));
                }

                if (gArgs.SoftSetBoolArg("-rescan", true)) {
                    LogPrintf("%s: parameter interaction: -salvagewallet=1 -> setting -rescan=1\n", __func__);
                }
            }

    -   处理 `-zapwallettxes`、`-persistmempool` 参数。

        `-zapwallettxes` 参数表示删除所有钱包交易，并且仅在启动时通过 `-rescan` 恢复区块链相关的那些部分（1 =保留tx元数据，例如账户所有者和支付请求信息，2 =丢弃tx元数据）。它暗示了在启动时删除交易池中的那些交易。同时，它也暗示了进行区块链扫描，即不能是多钱包。

            bool zapwallettxes = gArgs.GetBoolArg("-zapwallettxes", false);
            if (zapwallettxes && gArgs.SoftSetBoolArg("-persistmempool", false)) {
                LogPrintf("%s: parameter interaction: -zapwallettxes enabled -> setting -persistmempool=0\n", __func__);
            }

            if (zapwallettxes) {
                if (is_multiwallet) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-zapwallettxes"));
                }
                if (gArgs.SoftSetBoolArg("-rescan", true)) {
                    LogPrintf("%s: parameter interaction: -zapwallettxes enabled -> setting -rescan=1\n", __func__);
                }
            }      

    -   检查是否指定了 `-upgradewallet` 参数。

        在多钱包情况下，不能进行钱包升级。

            if (is_multiwallet) {
                if (gArgs.GetBoolArg("-upgradewallet", false)) {
                    return InitError(strprintf("%s is only allowed with a single wallet file", "-upgradewallet"));
                }
            }

    -   检查是否指定了 `-sysperms` 参数。

        如果指定了 `-sysperms` 参数，则抛出初始异常错误。这个参数表示了用系统默认的权限创建一个新文件，而不是 `077`，只有在禁用钱包功能的情况下才有效。

            if (gArgs.GetBoolArg("-sysperms", false))
                return InitError("-sysperms is not allowed in combination with enabled wallet functionality");

    -   检查是否同时指定了 `-prune`、`-rescan` 参数。

        在指定了修剪模式的情况下，不能执行扫描区块链的动作。所以如果同时指定了这两个参数，抛出错误。

            if (gArgs.GetArg("-prune", 0) && gArgs.GetBoolArg("-rescan", false))
                return InitError(_("Rescans are not possible in pruned mode. You will need to use -reindex which will download the whole blockchain again."));

    -   处理 `-maxtxfee` 参数。

        `-maxtxfee` 参数表示了在单个钱包的交易或原始交易中使用的最高总费用。如果设置过小，可能会中止大型交易。这个值不能小于最小中继交易费用。

            if (gArgs.IsArgSet("-maxtxfee"))
            {
                CAmount nMaxFee = 0;
                if (!ParseMoney(gArgs.GetArg("-maxtxfee", ""), nMaxFee))
                    return InitError(AmountErrMsg("maxtxfee", gArgs.GetArg("-maxtxfee", "")));
                if (nMaxFee > HIGH_MAX_TX_FEE)
                    InitWarning(_("-maxtxfee is set very high! Fees this large could be paid on a single transaction."));
                maxTxFee = nMaxFee;
                if (CFeeRate(maxTxFee, 1000) < ::minRelayTxFee)
                {
                    return InitError(strprintf(_("Invalid amount for -maxtxfee=<amount>: '%s' (must be at least the minrelay fee of %s to prevent stuck transactions)"),
                                               gArgs.GetArg("-maxtxfee", ""), ::minRelayTxFee.ToString()));
                }
            }

22. 获取 `-permitbaremultisig`、`-datacarrier`、`-datacarriersize`等参数。

        fIsBareMultisigStd = gArgs.GetBoolArg("-permitbaremultisig", DEFAULT_PERMIT_BAREMULTISIG);
        fAcceptDatacarrier = gArgs.GetBoolArg("-datacarrier", DEFAULT_ACCEPT_DATACARRIER);
        nMaxDatacarrierBytes = gArgs.GetArg("-datacarriersize", nMaxDatacarrierBytes);

23. 调用 `-SetMockTime` 方法，设置模拟时间。

        SetMockTime(gArgs.GetArg("-mocktime", 0)); 

24. 根据 `-peerbloomfilters` 参数，设置本地支持的服务。

        if (gArgs.GetBoolArg("-peerbloomfilters", DEFAULT_PEERBLOOMFILTERS))
            nLocalServices = ServiceFlags(nLocalServices | NODE_BLOOM);

25. 检测 `-rpcserialversion` 参数是否小于0，是否大于1。

        if (gArgs.GetArg("-rpcserialversion", DEFAULT_RPC_SERIALIZE_VERSION) < 0)
            return InitError("rpcserialversion must be non-negative.");

        if (gArgs.GetArg("-rpcserialversion", DEFAULT_RPC_SERIALIZE_VERSION) > 1)
            return InitError("unknown rpcserialversion requested.");

26. 获取 `-maxtipage` 参数值。

        nMaxTipAge = gArgs.GetArg("-maxtipage", DEFAULT_MAX_TIP_AGE);

27. 处理 `-mempoolreplacement` 参数。

    `-mempoolreplacement` 参数表示是否启用交易池交易替换。

        fEnableReplacement = gArgs.GetBoolArg("-mempoolreplacement", DEFAULT_ENABLE_REPLACEMENT);
        if ((!fEnableReplacement) && gArgs.IsArgSet("-mempoolreplacement")) {
            std::string strReplacementModeList = gArgs.GetArg("-mempoolreplacement", "");  // default is impossible
            std::vector<std::string> vstrReplacementModes;
            boost::split(vstrReplacementModes, strReplacementModeList, boost::is_any_of(","));
            fEnableReplacement = (std::find(vstrReplacementModes.begin(), vstrReplacementModes.end(), "fee") != vstrReplacementModes.end());
        }

28. 处理 `-vbparams` 参数。

    `-vbparams` 参数代表了对于指定的版本位部署，使用给定的开始/结束时间。

    重载版本位只在回归测试模式下才允许，否则会抛出初始异常错误。在回归测试模式下检查所有指定的版本位。

        if (gArgs.IsArgSet("-vbparams")) {
            // Allow overriding version bits parameters for testing
            if (!chainparams.MineBlocksOnDemand()) {
                return InitError("Version bits parameters may only be overridden on regtest.");
            }
            for (const std::string& strDeployment : gArgs.GetArgs("-vbparams")) {
                std::vector<std::string> vDeploymentParams;
                boost::split(vDeploymentParams, strDeployment, boost::is_any_of(":"));
                if (vDeploymentParams.size() != 3) {
                    return InitError("Version bits parameters malformed, expecting deployment:start:end");
                }
                int64_t nStartTime, nTimeout;
                if (!ParseInt64(vDeploymentParams[1], &nStartTime)) {
                    return InitError(strprintf("Invalid nStartTime (%s)", vDeploymentParams[1]));
                }
                if (!ParseInt64(vDeploymentParams[2], &nTimeout)) {
                    return InitError(strprintf("Invalid nTimeout (%s)", vDeploymentParams[2]));
                }
                bool found = false;
                for (int j=0; j<(int)Consensus::MAX_VERSION_BITS_DEPLOYMENTS; ++j)
                {
                    if (vDeploymentParams[0].compare(VersionBitsDeploymentInfo[j].name) == 0) {
                        UpdateVersionBitsParameters(Consensus::DeploymentPos(j), nStartTime, nTimeout);
                        found = true;
                        LogPrintf("Setting version bits activation parameters for %s to start=%ld, timeout=%ld\n", vDeploymentParams[0], nStartTime, nTimeout);
                        break;
                    }
                }
                if (!found) {
                    return InitError(strprintf("Invalid deployment (%s)", vDeploymentParams[0]));
                }
            }
        }
