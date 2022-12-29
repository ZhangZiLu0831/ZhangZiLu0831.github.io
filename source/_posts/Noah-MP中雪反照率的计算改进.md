---
title: Noah MP中雪反照率的计算改进
date: 2022-11-14 15:48:21
tags:
- WRF
- 数值模式
- 陆面模式
- Noah MP
categories: 垃圾笔记
math: true
---
# Noah-MP简介
Noah-MP是以Noah-LSM为基础发展的一种多层模型，相对于Noah-LSM，它对于下垫面的冠层、土壤、积雪有了更多的拓展。

Noah-MP允许3层雪，对于雪的模拟有了极大的提升，同时，Polar-WRF中对Noah LSM和Noah MP中海冰过程进行了改进，使得其在极地区域的能量模拟有了较大改善。
雪是重要的地表参数，尤其在积雪冰川常年覆盖的极地区域，将Polar-WRF中雪反照率的修改、订正十分必要。

![Noah-MP示意图](https://www.jsg.utexas.edu/noah-mp/files/webschematic4.jpg "Noah MP")

# Noah-MP与WRF的耦合
Noah-Mp主要通过读取MPTABLE.TBL读取所需参数(default)，主要程序由三部分组成：

1. 计算各冠层地表过程（径流、土壤湿度、雪过程、反照率与能量平衡等） 的程序：module_sf_noahmpdrv.F
2. 计算冰川表面能量的module_sf_noahmp_glacier.F.
3. 调用noah-mp的程序的module_surface_driver。
   
**noahmp_glacier.f**程序的使用与WPS输入的静态地理数据有关，当土地分类为Snow or ice时使用。
在Noah-mp中，**默认的土地并不区分冰雪，而是根据输入的雪深等参数进行计算调整，SNOWH**。
## MPTABLE.TBL
MPTABLE.TBL中给出了Noah-MP计算所需的参数，其中与积雪相关的参数如下
```
&noahmp_rad_parameters
 !------------------------------------------------------------------------------
 !                1       2       3       4       5       6       7       8     soil color index for soil albedo
 !------------------------------------------------------------------------------
 ALBSAT_VIS =   0.15,   0.11,   0.10,   0.09,   0.08,   0.07,   0.06,   0.05   ! saturated soil albedos
 ALBSAT_NIR =   0.30,   0.22,   0.20,   0.18,   0.16,   0.14,   0.12,   0.10   ! saturated soil albedos
 ALBDRY_VIS =   0.27,   0.22,   0.20,   0.18,   0.16,   0.14,   0.12,   0.10   ! dry soil albedos
 ALBDRY_NIR =   0.54,   0.44,   0.40,   0.36,   0.32,   0.28,   0.24,   0.20   ! dry soil albedos
 ALBICE     =   0.80,   0.55                                                   ! albedo land ice: 1=vis, 2=nir
 ALBLAK     =   0.60,   0.40                                                   ! albedo frozen lakes: 1=vis, 2=nir
 OMEGAS     =   0.8 ,   0.4                                                    ! two-stream parameter omega for snow
 BETADS     =   0.5                                                            ! two-stream parameter betad for snow
 BETAIS     =   0.5                                                            ! two-stream parameter betaI for snow
 EG         =   0.97,   0.98                                                   ! emissivity soil surface 1-soil;2-lake

/

&noahmp_global_parameters
…
! adjustable parameters for snow processes

  Z0SNO  = 0.002        !snow surface roughness length (m) (0.002)
  SSI    = 0.03         !liquid water holding capacity for snowpack (m3/m3) (0.03)
  SNOW_RET_FAC  = 5.e-5 !snowpack water release timescale factor (1/s)
  SNOW_EMIS     = 0.95  !snow emissivity (bring from hard-coded value of 1.0 to here)
  SWEMX  = 1.00         !new snow mass to fully cover old snow (mm)
                        !equivalent to 10mm depth (density = 100 kg/m3)
  TAU0          = 1.e6  !tau0 from Yang97 eqn. 10a
  GRAIN_GROWTH  = 5000. !growth from vapor diffusion Yang97 eqn. 10b
  EXTRA_GROWTH  = 10.   !extra growth near freezing Yang97 eqn. 10c
  DIRT_SOOT     = 0.3   !dirt and soot term Yang97 eqn. 10d
  BATS_COSZ     = 2.0   !zenith angle snow albedo adjustment; b in Yang97 eqn. 15
  BATS_VIS_NEW  = 0.95  !new snow visible albedo
  BATS_NIR_NEW  = 0.65  !new snow NIR albedo
  BATS_VIS_AGE  = 0.2   !age factor for diffuse visible snow albedo Yang97 eqn. 17
  BATS_NIR_AGE  = 0.5   !age factor for diffuse NIR snow albedo Yang97 eqn. 18
  BATS_VIS_DIR  = 0.4   !cosz factor for direct visible snow albedo Yang97 eqn. 15
  BATS_NIR_DIR  = 0.4   !cosz factor for direct NIR snow albedo Yang97 eqn. 16
  RSURF_SNOW = 50.0     !surface resistence for snow [s/m]
  RSURF_EXP = 5.0       !exponent in the shape parameter for soil resistance option 1

/
```
通过修改MPTABLE.TBL中的雪过程参数可以对参数化方案进行优化。
## Noah-MP中的雪反照率方案
Noah-MP LSM包括单独的冰川处理和改进的雪物理方案，在积雪最多可达三层，该方法还采用了一种改进的双流辐射传输方案，考虑了三维冠层结构，计算植被反射、吸收和传输的辐射通量。该LSM使用“平铺”(tile)方法计算反照率，这是能源预算中的一个关键因素，考虑到裸露的地面、植被冠层和积雪。
对于北极地区，土地利用类型对于反照率的影响较小，在地面上，主要考虑冰川与积雪的反照率。
1. CLASS方案
   
   CLASS方案的公式计算如下：
   $$
   \alpha_1=0.55+(\alpha_{old}-0.55 )\exp^{\frac{-0.01dt}{3600}} \tag{1}
   $$

   $$
   \mathcal f_{sn}=tanh(\frac{h_{sn}}{2.5_{z_0}(\frac{\rho_{sn}}{\rho_{new}})^{\mathcal{f}_m}}) \tag{2}
   $$

   $$
   \alpha_s=\alpha_{1}+\mathcal f_{sn}(0.84-\alpha_1) \tag{3}
   $$
   
   $$
   \alpha_{sd1}=\alpha_{sd2} =\alpha_{si1}=\alpha_{si2}\tag{4}
   $$
其中，$\alpha_{old}$为最后时间步长(dt)反照率，$\mathcal f_{sn}$为积雪量（分数表达），$h_{sn}$为积雪深度,$\rho_{sn}$积雪密度,$\rho_{new}$新雪密度，设置为$100kg/m^{-3}$，$\mathcal{f}_m$为融化因子，默认值为1.0，$\alpha_{sd1},\alpha_{sd2} ,\alpha_{si1},\alpha_{si2}$分别为可见光、近红外波段的直接和漫反照率。

CLASS方案相对简单，由用户定义的衰减因子调制的指数速率降低雪反照率。该方法试图用一个时间常数参数隐含地解释影响雪反照率衰减的一系列因素，运算效率较高，却并不能较好地反映冰雪反照率的复杂过程。

代码中CLASS方案的函数：
```
!== begin snowalb_class ============================================================================

  SUBROUTINE SNOWALB_CLASS (parameters,NBAND,QSNOW,DT,ALB,ALBOLD,ALBSND,ALBSNI,ILOC,JLOC)
! ----------------------------------------------------------------------
  IMPLICIT NONE
! --------------------------------------------------------------------------------------------------
! input

  type (noahmp_parameters), intent(in) :: parameters
  INTEGER,INTENT(IN) :: ILOC !grid index
  INTEGER,INTENT(IN) :: JLOC !grid index
  INTEGER,INTENT(IN) :: NBAND  !number of waveband classes

  REAL,INTENT(IN) :: QSNOW     !snowfall (mm/s)
  REAL,INTENT(IN) :: DT        !time step (sec)
  REAL,INTENT(IN) :: ALBOLD    !snow albedo at last time step

! in & out

  REAL,                INTENT(INOUT) :: ALB        ! 
! output

  REAL, DIMENSION(1:2),INTENT(OUT) :: ALBSND !snow albedo for direct(1=vis, 2=nir)
  REAL, DIMENSION(1:2),INTENT(OUT) :: ALBSNI !snow albedo for diffuse
! ---------------------------------------------------------------------------------------------

! ------------------------ local variables ----------------------------------------------------
  INTEGER :: IB          !waveband class

! ---------------------------------------------------------------------------------------------
! zero albedos for all points

        ALBSND(1: NBAND) = 0.
        ALBSNI(1: NBAND) = 0.

! when cosz > 0

         ALB = 0.55 + (ALBOLD-0.55) * EXP(-0.01*DT/3600.)

! 1 mm fresh snow(SWE) -- 10mm snow depth, assumed the fresh snow density 100kg/m3
! here assume 1cm snow depth will fully cover the old snow

         IF (QSNOW > 0.) then
           ALB = ALB + MIN(QSNOW,parameters%SWEMX/DT) * (0.84-ALB)/(parameters%SWEMX/DT)
         ENDIF

         ALBSNI(1)= ALB         ! vis diffuse
         ALBSNI(2)= ALB         ! nir diffuse
         ALBSND(1)= ALB         ! vis direct
         ALBSND(2)= ALB         ! nir direct

  END SUBROUTINE SNOWALB_CLASS
```
2. BATS方案
   
   生物圈-大气转移方案(BATS)地面雪反照率公式从Wiscombe和Warren(1980)的积雪辐射转移计算中推断出来的。是一种中等复杂算法，它允许较高的计算效率，同时保持可见光和近红外光谱上直接和漫反射率与雪龄、表面温度、太阳照明角和吸收杂质之间的上述相互作用。

   由于BATS算法在复杂性与计算效率的有着较好的平衡，该算法有着广泛地应用。在Noah-MP中，雪反照率的计算可由如下公式概括：
   $$
   \Z_c=\frac{1.5}{1+cosZ}-0.5 \tag{1}
   $$
   $$
   \alpha_{si1}=0.95(1-0.2A_c) \tag{2}
   $$
   $$
   \alpha_{si2}=0.65(1-0.5A_c) \tag{3}
   $$
   $$
   \alpha_{sd1}=\alpha_{si1}+0.4Z_c(1-\alpha_{si1}) \tag{4}
   $$
   $$
   \alpha_{sd2}=\alpha_{si2}+0.4Z_c(1-\alpha_{si2}) \tag{5}
   $$
Z为太阳高度角，$A_c$为雪龄。
雪龄由以下公式计算：
$$
   A_c=\frac{\tau_s}{1+\tau_s} \tag{1}
$$
$$
\tau_s^t=\tau_s^{t-1}[1-max(0,\Delta SWE)/SWE_{MX}] \tag{2}
$$
$$
\Delta\tau_s=(r_1+r_2+r_3)\frac{\Delta t}{\tau_0} \tag{3}
$$
$$
\begin{cases}
arg=5000(\cfrac{1}{TFRZ}-\cfrac{1}{TG} )\\
r_1=\exp(arg)\\
r_2=min(1,\exp(10×arg))\\
r_3=0.3
\end{cases} \tag{4}
$$
TFRZ为冰冻温度，设为273.16K，TG为地表温度。
根据上述描述，我们可以总结出BATS算法中可以调整的参数并进行修改。

相关代码如下，少许中文注释为自行添加：
```
! 一些BATS参数，从MPTABLE中读取
!------------------------------------------------------------------------------------------!
! From the rad section of MPTABLE.TBL
!------------------------------------------------------------------------------------------!

     REAL :: ALBSAT(MBAND)       !saturated soil albedos: 1=vis, 2=nir
     REAL :: ALBDRY(MBAND)       !dry soil albedos: 1=vis, 2=nir
     REAL :: ALBICE(MBAND)       !albedo land ice: 1=vis, 2=nir
     REAL :: ALBLAK(MBAND)       !albedo frozen lakes: 1=vis, 2=nir
     REAL :: OMEGAS(MBAND)       !two-stream parameter omega for snow
     REAL :: BETADS              !two-stream parameter betad for snow
     REAL :: BETAIS              !two-stream parameter betad for snow
     REAL :: EG(2)               !emissivity

!------------------------------------------------------------------------------------------!
! From the globals section of MPTABLE.TBL
!------------------------------------------------------------------------------------------!
REAL :: Z0SNO        !snow surface roughness length (m) (0.002)
REAL :: SSI          !liquid water holding capacity for snowpack (m3/m3)
REAL :: SNOW_RET_FAC !snowpack water release timescale factor (1/s)
REAL :: SNOW_EMIS    !snow emissivity
REAL :: SWEMX        !new snow mass to fully cover old snow (mm)
REAL :: TAU0         !tau0 from Yang97 eqn. 10a
REAL :: GRAIN_GROWTH !growth from vapor diffusion Yang97 eqn. 10b
REAL :: EXTRA_GROWTH !extra growth near freezing Yang97 eqn. 10c
REAL :: BATS_COSZ    !zenith angle snow albedo adjustment; b in Yang97 eqn. 15
REAL :: BATS_VIS_NEW !new snow visible albedo
REAL :: BATS_NIR_NEW !new snow NIR albedo
REAL :: BATS_VIS_AGE !age factor for diffuse visible snow albedo Yang97 eqn. 17
REAL :: BATS_NIR_AGE !age factor for diffuse NIR snow albedo Yang97 eqn. 18
REAL :: BATS_VIS_DIR !cosz factor for direct visible snow albedo Yang97 eqn. 15
REAL :: BATS_NIR_DIR !cosz factor for direct NIR snow albedo Yang97 eqn. 16
REAL :: RSURF_SNOW   !surface resistance for snow(s/m)
REAL :: DIRT_SOOT    !dirt and soot term Yang97 eqn. 10d

！开始辐射计算，该函数主要由计算反照率的ALBEDO函数和计算各太阳辐射的函数SURRAD的调用结合
!== begin radiation ================================================================================

  SUBROUTINE RADIATION (parameters,VEGTYP  ,IST     ,ICE     ,NSOIL   , & !in
                        SNEQVO  ,SNEQV   ,DT      ,COSZ    ,SNOWH   , & !in
                        TG      ,TV      ,FSNO    ,QSNOW   ,FWET    , & !in
                        ELAI    ,ESAI    ,SMC     ,SOLAD   ,SOLAI   , & !in
                        FVEG    ,ILOC    ,JLOC    ,                   & !in
                        ALBOLD  ,TAUSS   ,                            & !inout
                        FSUN    ,LAISUN  ,LAISHA  ,PARSUN  ,PARSHA  , & !out
                        SAV     ,SAG     ,FSR     ,FSA     ,FSRV    , &
                        FSRG    ,ALBSND  ,ALBSNI  ,BGAP    ,WGAP    )   !out
！以下省略参数说明部分
    CALL(ALBEDO) ！调用函数计算反照率
    CALL SURRAD  ！根据ALBEDO函数计算结果，计算太阳辐射各分量
  END SUBROUTINE RADIATION
!== begin albedo ===================================================================================

  SUBROUTINE ALBEDO (parameters,VEGTYP ,IST    ,ICE    ,NSOIL  , & !in
                     DT     ,COSZ   ,FAGE   ,ELAI   ,ESAI   , & !in
                     TG     ,TV     ,SNOWH  ,FSNO   ,FWET   , & !in
                     SMC    ,SNEQVO ,SNEQV  ,QSNOW  ,FVEG   , & !in
                     ILOC   ,JLOC   ,                         & !in
                     ALBOLD ,TAUSS                          , & !inout
                     ALBGRD ,ALBGRI ,ALBD   ,ALBI   ,FABD   , & !out
                     FABI   ,FTDD   ,FTID   ,FTII   ,FSUN   , & !out
                     FREVI  ,FREVD  ,FREGD  ,FREGI  ,BGAP   , & !out
                     WGAP   ,ALBSND ,ALBSNI )

! --------------------------------------------------------------------------------------------------
! surface albedos. also fluxes (per unit incoming direct and diffuse
! radiation) reflected, transmitted, and absorbed by vegetation.
! also sunlit fraction of the canopy.
! --------------------------------------------------------------------------------------------------
END SUBROUTINE ALBEDO

! 计算雪龄
!== begin snow_age =================================================================================

  SUBROUTINE SNOW_AGE (parameters,DT,TG,SNEQVO,SNEQV,TAUSS,FAGE)
! ----------------------------------------------------------------------
  IMPLICIT NONE
! ------------------------ code history ------------------------------------------------------------
! from BATS
! ------------------------ input/output variables --------------------------------------------------
!input
  type (noahmp_parameters), intent(in) :: parameters
   REAL, INTENT(IN) :: DT        !main time step (s)
   REAL, INTENT(IN) :: TG        !ground temperature (k)
   REAL, INTENT(IN) :: SNEQVO    !snow mass at last time step(mm)
   REAL, INTENT(IN) :: SNEQV     !snow water per unit ground area (mm)

!output
   REAL, INTENT(OUT) :: FAGE     !snow age

!input/output
   REAL, INTENT(INOUT) :: TAUSS      !non-dimensional snow age
!local
   REAL            :: TAGE       !total aging effects
   REAL            :: AGE1       !effects of grain growth due to vapor diffusion
   REAL            :: AGE2       !effects of grain growth at freezing of melt water
   REAL            :: AGE3       !effects of soot
   REAL            :: DELA       !temporary variable
   REAL            :: SGE        !temporary variable
   REAL            :: DELS       !temporary variable
   REAL            :: DELA0      !temporary variable
   REAL            :: ARG        !temporary variable
! See Yang et al. (1997) J.of Climate for detail.
!---------------------------------------------------------------------------------------------------

   IF(SNEQV.LE.0.0) THEN
          TAUSS = 0.
   ELSE
          DELA0 = DT/parameters%TAU0
          ARG   = parameters%GRAIN_GROWTH*(1./TFRZ-1./TG)
          AGE1  = EXP(ARG)
          AGE2  = EXP(AMIN1(0.,parameters%EXTRA_GROWTH*ARG))
          AGE3  = parameters%DIRT_SOOT
          TAGE  = AGE1+AGE2+AGE3
          DELA  = DELA0*TAGE
          DELS  = AMAX1(0.0,SNEQV-SNEQVO) / parameters%SWEMX
          SGE   = (TAUSS+DELA)*(1.0-DELS)
          TAUSS = AMAX1(0.,SGE)
   ENDIF

   FAGE= TAUSS/(TAUSS+1.)

  END SUBROUTINE SNOW_AGE
!== begin snowalb_bats =============================================================================

  SUBROUTINE SNOWALB_BATS (parameters,NBAND,FSNO,COSZ,FAGE,ALBSND,ALBSNI)
! --------------------------------------------------------------------------------------------------
  IMPLICIT NONE
! --------------------------------------------------------------------------------------------------
! input

  type (noahmp_parameters), intent(in) :: parameters
  INTEGER,INTENT(IN) :: NBAND  !number of waveband classes

  REAL,INTENT(IN) :: COSZ    !cosine solar zenith angle
  REAL,INTENT(IN) :: FSNO    !snow cover fraction (-)
  REAL,INTENT(IN) :: FAGE    !snow age correction

! output

  REAL, DIMENSION(1:2),INTENT(OUT) :: ALBSND !snow albedo for direct(1=vis, 2=nir)
  REAL, DIMENSION(1:2),INTENT(OUT) :: ALBSNI !snow albedo for diffuse
! ---------------------------------------------------------------------------------------------

! ------------------------ local variables ----------------------------------------------------
  INTEGER :: IB          !waveband class

  REAL :: FZEN                 !zenith angle correction
  REAL :: CF1                  !temperary variable
  REAL :: SL2                  !2.*SL
  REAL :: SL1                  !1/SL
  REAL :: SL                   !adjustable parameter
!  REAL, PARAMETER :: C1 = 0.2  !default in BATS 
!  REAL, PARAMETER :: C2 = 0.5  !default in BATS
!  REAL, PARAMETER :: C1 = 0.2 * 2. ! double the default to match Sleepers River's
!  REAL, PARAMETER :: C2 = 0.5 * 2. ! snow surface albedo (double aging effects)
! ---------------------------------------------------------------------------------------------
! zero albedos for all points

        ALBSND(1: NBAND) = 0.
        ALBSNI(1: NBAND) = 0.

! when cosz > 0

        SL=parameters%BATS_COSZ
        SL1=1./SL
        SL2=2.*SL
        CF1=((1.+SL1)/(1.+SL2*COSZ)-SL1)
        FZEN=AMAX1(CF1,0.)

        ALBSNI(1)=parameters%BATS_VIS_NEW*(1.-parameters%BATS_VIS_AGE*FAGE)         
        ALBSNI(2)=parameters%BATS_NIR_NEW*(1.-parameters%BATS_NIR_AGE*FAGE)        

        ALBSND(1)=ALBSNI(1)+parameters%BATS_VIS_DIR*FZEN*(1.-ALBSNI(1))    !  vis direct
        ALBSND(2)=ALBSNI(2)+parameters%BATS_VIS_DIR*FZEN*(1.-ALBSNI(2))    !  nir direct

  END SUBROUTINE SNOWALB_BATS
！地表反照率，考虑地面类型，雪反照率如何应用至地面中
!== begin groundalb ================================================================================

  SUBROUTINE GROUNDALB (parameters,NSOIL   ,NBAND   ,ICE     ,IST     , & !in
                        FSNO    ,SMC     ,ALBSND  ,ALBSNI  ,COSZ    , & !in
                        TG      ,ILOC    ,JLOC    ,                   & !in
                        ALBGRD  ,ALBGRI  )                              !out
…！省略输入参数说明
!output

  REAL, DIMENSION(1:    2), INTENT(OUT) :: ALBGRD !ground albedo (direct beam: vis, nir)
  REAL, DIMENSION(1:    2), INTENT(OUT) :: ALBGRI !ground albedo (diffuse: vis, nir)

!local 

  INTEGER                               :: IB     !waveband number (1=vis, 2=nir)
  REAL                                  :: INC    !soil water correction factor for soil albedo
  REAL                                  :: ALBSOD !soil albedo (direct)
  REAL                                  :: ALBSOI !soil albedo (diffuse)
! --------------------------------------------------------------------------------------------------

  DO IB = 1, NBAND
        INC = MAX(0.11-0.40*SMC(1), 0.)
        IF (IST .EQ. 1)  THEN                     !soil
           ALBSOD = MIN(parameters%ALBSAT(IB)+INC,parameters%ALBDRY(IB))
           ALBSOI = ALBSOD
        ELSE IF (TG .GT. TFRZ) THEN               !unfrozen lake, wetland
           ALBSOD = 0.06/(MAX(0.01,COSZ)**1.7 + 0.15)
           ALBSOI = 0.06
        ELSE                                      !frozen lake, wetland
           ALBSOD = parameters%ALBLAK(IB)
           ALBSOI = ALBSOD
        END IF

! increase desert and semi-desert albedos

!        IF (IST .EQ. 1 .AND. ISC .EQ. 9) THEN
!           ALBSOD = ALBSOD + 0.10
!           ALBSOI = ALBSOI + 0.10
!        end if

        ALBGRD(IB) = ALBSOD*(1.-FSNO) + ALBSND(IB)*FSNO
        ALBGRI(IB) = ALBSOI*(1.-FSNO) + ALBSNI(IB)*FSNO
  END DO
！地表反照率与地面积雪量FSNO有关,FSNO与输入的雪水当量、雪深变量有关
  END SUBROUTINE GROUNDALB
！FSNO的计算
! ground snow cover fraction [Niu and Yang, 2007, JGR]

     FSNO = 0.
     IF(SNOWH.GT.0.)  THEN
         BDSNO    = SNEQV / SNOWH
         FMELT    = (BDSNO/100.)**parameters%MFSNO
         !FSNO     = TANH( SNOWH /(2.5* Z0 * FMELT))
         FSNO     = TANH( SNOWH /(parameters%SCFFAC * FMELT)) ! C.He: bring hard-coded 2.5*z0 to MPTABLE tunable parameter SCFFAC
     ENDIF

```
综上，Noah-MP中的雪过程与反照率与输入的默认参数与场有关。
可修改：默认的MPTABLE与NpahLSM.F中，列出的与雪反照率相关的参数，见表1.
WPS输入的场：输入的ERA5的雪深等。
以上只考虑了地表过程，而Polar-WRF中，对于海冰的能量同样值得注意（对海冰场的修改）。
接下来将考虑与SNICAR模型结合，将SNICAR反照率反馈输入至polar-WRF中。
# 与SNICAR结合
如前文代码描述，WRF-Noah-NP中，在有雪时地表的反照率计算如下：

$
\alpha_{ground}=\alpha_{snow}*SCF+(1-SCF)*\alpha_{soil}
$

SCF即为积雪覆盖率，在代码中为FSNO，相关代码参见上文。
如前所言，Noah-MP中对于雪反照率的参数方案主要与雪龄与太阳高度角相关，然而该方案对于积雪的融化与雪粒、雪形态等微物理过程的考虑较为欠缺。此外，对于雪中吸收性粒子对雪反照率的影响考虑并不彻底。
为此，我们考虑使用SNICAR模式，对Noah-MP中的雪物理过程进行修正。

SNICAR(the Snow Ice and Aerosol Radiative model)是一个多层耦合积雪老化和辐射转移方案，并被耦合至了CLM陆面模式中，并被WRF使用，它利用Mie理论计算了冰粒和吸光杂质的光学性质，辐射传输模型采用的五波段光谱网格。冰的光学性质取决于雪的有效晶粒尺寸，而这些都被SNICAR中雪有效粒径考虑在内。
## SNICAR对雪反照率的计算
SNICAR根据Legagneux等人[2004]的经验方程计算了代表雪老化过程的有效雪粒半径,这是一个与雪温$T_{snow}$、雪层间温度梯度$TG_{snow}$和雪密度$\rho_{snow}$的函数。公式为：
$$
r_e(t)=[r_e(t-1)+\frac{dr_e}{dt}_{dry}dt]f_{old}+r_{e,refreeze}f_{refreeze}+r_{e,fresh_snow}f_{fresh_snow} \tag{1}
$$
$$
\frac{dr_e}{dt}_{dry}=\frac{dr_e}{dt}_{0}\frac{\tau}{dr_{fresh_snow}+\tau}^{\frac{1}{\kappa}} \tag{2}
$$
$$
\frac{dr_e}{dt}_{wet}=\frac{10^{18}C_1f^3_{liq}}{4\pi r_e^2} \tag{3}
$$
SNICAR模式将雪分为五层，并考虑了吸收性粒子的粒径、老化、形态以及雪融化效率等，上述公式（1）给出了SNICAR中雪有效粒径的计算公式，公式（2）和（3）则为干变质作用和湿变质作用引起的雪半径的变质。

$r_{e,fresh_snow}$设为恒定值54μm;τ和κ是离线计算的最佳拟合参数，并从查找表中提取，查找表基于雪温度$T_{snow}$、雪温度梯度$TG_{snow}$、雪密度$\rho_{snow}$;C1为Brun[1989]的湿雪老化常数，$f_{liq}$为雪中的液态水分数，$r_{e,refreeze}$设置为1000 μm, $f_{refreeze}$为再冻层质量的百分比，$f_{new_snow}$为新雪层质量的百分比。

SNICAR模式可以较为准确的描述雪层的辐射传输特性与垂直结构，非常适合研究吸收性杂杂质对于积雪反照率的影响，同时对与陆地冰川、积雪也有较好地区分，然而，SNICAR模式对于计算的成本要求过高，不适用于长时间序列或高分辨率模拟。为此，考虑将SNICAR模式与Noah-MP耦合，改进Noah-MP的雪反照率过程，同时保留Polar-WRF中对于Noah-MP对于海冰能量平衡的计算。
## SNICAR与Noah-MP的耦合
将SNICAR辐射传输方案的输出的雪反照率替代Noah-MP BATS计算的雪反照率，SNICAR模式的输入可通过WRF/Noah-MP的输出与观测得到。

<center><img src="https://alpha.glilmu.com/i/2022/12/05/txpqcz.jpg
" width="700" > SNICAR与WRF/Noah-MP耦合示意图 </center>

# 总结与参考文献

我们可以将当前对于积雪反照率的改进总结为3种方式：
1. **对输入的静态地理数据的改进。**
   
   主要修改输入的土地利用类型、植被覆盖与背景反照率
2. **对雪反照率过程参数的改进**
   
   主要修改雪反照率过程的参数，一般根据观测或高分辨率卫星数据修改。
3. **引入或改进新的雪反照率方案，并将新雪反照率方案应用至WRF/Noah-MP中**

其中第1种在目前研究中主要是以辅助或者研究植被冠层的影响为主，而对2、3种方式而言，其改进都要基于现场观测数据的基础之上的，或者至少要用现场观测数据来作为验证，无论如何，当前情况下，观测与模拟的结合都是当前研究的趋势。通过观测修正模式中的Default，对模式模拟性能进行改进，又通过观测来验证；同时，被改进后的模式被应用至整个研究区域，用来分析研究区域气象、气候因素变化的原因，也降低了气候模拟的不确定性。

作为一个复杂混沌系统，将来，气象与气候的研究趋势将会向多尺度、多圈层、多源数据的结合与应用上。

**参考文献**
1. Liu, L.; Menenti, M.; Ma, Y .
Evaluation of Albedo Schemes in
WRF Coupled with Noah-MP on the
Parlung No. 4 Glacier. Remote Sens.
2022, 14, 3934. https://doi.org/
10.3390/rs14163934
2. Oaida, C. M., Y. Xue, M. G. Flanner,
S. M. Skiles, F. De Sales, and T. H. Painter
(2015), Improving snow albedo
processes in WRF/SSiB regional climate
model to assess impact of dust and
black carbon in snow on surface energy
balance and hydrology over western
U.S., J. Geophys. Res. Atmos., 120,
3228–3248, doi:10.1002/2014JD022444.
3. Abolafia-Rosenzweig, R., He, C., 
McKenzie Skiles, S., Chen, F., & Gochis, 
D. (2022). Evaluation and optimization 
of snow albedo scheme in Noah-MP 
land surface model using in situ spectral 
observations in the Colorado Rockies. 
Journal of Advances in Modeling Earth 
Systems, 14, e2022MS003141. https://doi.
org/10.1029/2022MS003141
4. Wang, W., Yang, K., Zhao, L., Zheng, Z., Lu, H., Mamtimin, A., Ding, B., Li, X., Zhao, L., Li, H., Che, T., & Moore, J. C. (2020). Characterizing Surface Albedo of Shallow Fresh Snow and Its Importance for Snow Ablation on the Interior of the Tibetan Plateau, Journal of Hydrometeorology, 21(4), 815-827. Retrieved Dec 6, 2022, from https://journals.ametsoc.org/view/journals/hydr/21/4/jhm-d-19-0193.1.xml
5. lanner, M. G., Arnheim, J. B., Cook, J. M., Dang, C., He, C., Huang, X., Singh, D., Skiles, S. M., Whicker, C. A., and Zender, C. S.: SNICAR-ADv3: a community tool for modeling spectral snow albedo, Geosci. Model Dev., 14, 7673–7704, https://doi.org/10.5194/gmd-14-7673-2021, 2021.