# sat-http-example Function-call (Operation) Flow-chart

```C++
NS_LOG_COMPONENT_DEFINE ("sat-http-example");

int main (int argc, char *argv[])
{
  std::string scenario = "simple";
  double duration = 20; // double duration = 1000;
  SatHelper::PreDefinedScenario_t satScenario = SatHelper::SIMPLE; // Declare a variable satScenario that has enum type defined in class SatHelper

  Config::SetDefault ("ns3::SatHelper::ScenarioCreationTraceEnabled", BooleanValue (true));

  auto simulationHelper = CreateObject<SimulationHelper> ("example-http");
  Config::SetDefault ("ns3::SatEnvVariables::EnableSimulationOutputOverwrite", BooleanValue (true));

  // read command line parameters given by user
  CommandLine cmd;
  cmd.AddValue ("scenario", "Test scenario to use. (simple, larger or full)",
                scenario);
  cmd.AddValue ("duration", "Simulation duration (in seconds)",
                duration);
  simulationHelper->AddDefaultUiArguments (cmd);
  cmd.Parse (argc, argv);

  if (scenario == "larger")
    {
      satScenario = SatHelper::LARGER;
    }
  else if (scenario == "full")
    {
      satScenario = SatHelper::FULL;
    }

  /// Set simulation output details
  simulationHelper->SetSimulationTime (duration);
  simulationHelper->SetOutputTag (scenario);


  LogComponentEnableAll (LOG_PREFIX_ALL);
  LogComponentEnable ("HttpClient", LOG_LEVEL_ALL);
  LogComponentEnable ("HttpServer", LOG_LEVEL_ALL);
  //LogComponentEnable ("sat-http-example", LOG_LEVEL_ALL);

  // remove next line from comments to run real time simulation
  // GlobalValue::Bind ("SimulatorImplementationType", StringValue ("ns3::RealtimeSimulatorImpl"));

  // Creating the reference system. Note, currently the satellite module supports
  // only one reference system, which is named as "Scenario72". The string is utilized
  // in mapping the scenario to the needed reference system configuration files. Arbitrary
  // scenario name results in fatal error.
  Ptr<SatHelper> helper = simulationHelper->CreateSatScenario (satScenario);

  // get users
  NodeContainer utUsers = helper->GetUtUsers ();
  NodeContainer gwUsers = helper->GetGwUsers ();

  ThreeGppHttpHelper httpHelper;
  httpHelper.InstallUsingIpv4 (gwUsers.Get (0), utUsers);
  httpHelper.GetServer ().Start (Seconds (1.0));

  auto apps = httpHelper.GetClients ();
  apps.Start (Seconds (3.0));

  uint32_t i = 0;
  std::vector<Ptr<ClientRxTracePlot> > plots;
  for (auto app = apps.Begin (); app != apps.End (); app++, i++)
  {
    std::stringstream plotName;
    plotName << "3GPP-HTTP-client-" << i << "-trace";
  	plots.push_back (CreateObject<ClientRxTracePlot> (*app, plotName.str ()));
  }

  NS_LOG_INFO ("--- sat-http-example ---");
  NS_LOG_INFO ("  Scenario used: " << scenario);
  NS_LOG_INFO ("  ");

  simulationHelper->RunSimulation();
        
  plots.clear();

  return 0;

} // end of `int main (int argc, char *argv[])`

```
auto simulationHelper = CreateObject<SimulationHelper> ("example-http"); at line 14

```C++
template <typename T, typename T1>
Ptr<T> CreateObject (T1 a1)
{
  return CompleteConstruct (new T (a1));
}


```


```C++
void SatHelper::CreatePredefinedScenario (PreDefinedScenario_t scenario)
{
  NS_LOG_FUNCTION (this);

  switch (scenario)
    {
    case SIMPLE:
      CreateSimpleScenario ();
      break;

    case LARGER:
      CreateLargerScenario ();
      break;

    case FULL:
      CreateFullScenario ();
      break;

    default:
      NS_FATAL_ERROR ("Not supported predefined scenario.");
      break;
    }
}
```
* if `scenario = SIMPLE` then goto CreateSimpleScenario () *

```C++
void SatHelper::CreateSimpleScenario ()
{
  NS_LOG_FUNCTION (this);

  SatBeamUserInfo beamInfo = SatBeamUserInfo (1,1);
  BeamUserInfoMap_t beamUserInfos;
  beamUserInfos[8] = beamInfo;

  DoCreateScenario (beamUserInfos, 1);

  m_creationSummaryTrace ("*** Simple Scenario Creation Summary ***");
}
```
Every scenario calls SatHelper:DoCreateScenario(...)

```C++
void SatHelper::DoCreateScenario (BeamUserInfoMap_t& beamInfos, uint32_t gwUsers)
{
  NS_LOG_FUNCTION (this);

  if (m_scenarioCreated)
    {
      Singleton<SatLog>::Get ()->AddToLog (SatLog::LOG_WARNING, "", "Scenario tried to re-create with SatHelper. Creation can be done only once.");
    }
  else
    {
      SetNetworkAddresses (beamInfos, gwUsers);

      if (m_creationTraces)
        {
          EnableCreationTraces ();
        }

      InternetStackHelper internet;

      // create all possible GW nodes, set mobility to them and install to Internet
      NodeContainer gwNodes;
      gwNodes.Create (m_satConf->GetGwCount ());
      SetGwMobility (gwNodes);
      internet.Install (gwNodes);

      for ( BeamUserInfoMap_t::iterator info = beamInfos.begin (); info != beamInfos.end (); info++)
        {
          // create UTs of the beam, set mobility to them and install to Internet
          NodeContainer uts;
          uts.Create (info->second.GetUtCount ());
          SetUtMobility (uts, info->first);
          internet.Install (uts);

          for ( uint32_t i = 0; i < info->second.GetUtCount (); i++ )
            {
              // create and install needed users
              m_userHelper->InstallUt (uts.Get (i), info->second.GetUtUserCount (i));
            }

          std::vector<uint32_t> rtnConf = m_satConf->GetBeamConfiguration (info->first, SatEnums::LD_RETURN);
          std::vector<uint32_t> fwdConf = m_satConf->GetBeamConfiguration (info->first, SatEnums::LD_FORWARD);

          /**
           * GW and beam ids are assumed to be the same for both directions
           * currently!
           */
          NS_ASSERT (rtnConf[SatConf::GW_ID_INDEX] == fwdConf[SatConf::GW_ID_INDEX]);
          NS_ASSERT (rtnConf[SatConf::BEAM_ID_INDEX] == fwdConf[SatConf::BEAM_ID_INDEX]);

          // gw index starts from 1 and we have stored them starting from 0
          Ptr<Node> gwNode = gwNodes.Get (rtnConf[SatConf::GW_ID_INDEX] - 1);
          m_beamHelper->Install (uts,
                                 gwNode,
                                 rtnConf[SatConf::GW_ID_INDEX],
                                 rtnConf[SatConf::BEAM_ID_INDEX],
                                 rtnConf[SatConf::U_FREQ_ID_INDEX],
                                 rtnConf[SatConf::F_FREQ_ID_INDEX],
                                 fwdConf[SatConf::U_FREQ_ID_INDEX],
                                 fwdConf[SatConf::F_FREQ_ID_INDEX]);
        }

      m_userHelper->InstallGw (m_beamHelper->GetGwNodes (), gwUsers);

      if (m_packetTraces)
        {
          EnablePacketTrace ();
        }

      m_scenarioCreated = true;
    }

  m_beamHelper->Init ();
}
```

