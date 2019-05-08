---
layout: post
title:  "ORMs Are Not Easier"
date:   2019-05-08 00:00:00
tags:   scala slick orm sql
---

By now I've worked on enough production projects that used an ORM: ActiveRecord, Entity Framework, F# SQL Type Providers, and most recently Slick. They always disappoint in the end.

## What happened this time?

```
slick.SlickTreeException: Cannot convert node to SQL Comprehension
GroupBy t3 : Vector[(t3<(Option[String'], Option[String'])>, Vector[t4<{s5: Option[scala.math.BigDecimal'], s6: SOption[OptionDisc/Int], s7: Option[String'], s8: SOption[OptionDisc/Int], s9: Option[String']}>])]
  from s2: Bind : Vector[t4<{s5: Option[scala.math.BigDecimal'], s6: SOption[OptionDisc/Int], s7: Option[String'], s8: SOption[OptionDisc/Int], s9: Option[String']}>]
    from s10: Filter s11 : Vector[t12<{s13: String', s14: String', s15: String', s16: String', s17: String', s18: Option[String'], s19: Int', s20: Option[String'], s21: String', s22: scala.math.BigDecimal', s23: String', s24: String', s25: String', s26: String', s27: String', s28: Boolean', s29: scala.math.BigDecimal', s30: String', s31: String', s32: Boolean', s33: Option[String'], s34: String', s35: Option[String'], s36: Option[String'], s37: java.sql.Date', s38: Option[String'], s39: Option[String'], s40: String', s41: Option[String'], s42: String', s43: Option[scala.math.BigDecimal'], s44: scala.math.BigDecimal', s45: String', s46: Option[String'], s47: SOption[OptionDisc/Int], s48: Option[String'], s49: Option[String'], s50: Option[String'], s51: Option[String'], s52: Option[String'], s53: Option[String'], s54: Option[Int'], s55: Option[String'], s56: Option[String'], s57: Option[scala.math.BigDecimal'], s58: Option[String'], s59: Option[String'], s60: Option[String'], s61: Option[String'], s62: Option[String'], s63: Option[Boolean'], s64: Option[scala.math.BigDecimal'], s65: Option[String'], s66: Option[String'], s67: Option[Boolean'], s68: SOption[OptionDisc/Int], s69: Option[String'], s70: Option[String'], s71: Option[String'], s72: Option[String'], s73: Option[String'], s74: Option[String'], s75: Option[Int'], s76: Option[String'], s77: Option[String'], s78: Option[scala.math.BigDecimal'], s79: Option[String'], s80: Option[String'], s81: Option[String'], s82: Option[String'], s83: Option[String'], s84: Option[Boolean'], s85: Option[scala.math.BigDecimal'], s86: Option[String'], s87: Option[String'], s88: Option[Boolean']}>]
      from s11: Bind : Vector[t12<{s13: String', s14: String', s15: String', s16: String', s17: String', s18: Option[String'], s19: Int', s20: Option[String'], s21: String', s22: scala.math.BigDecimal', s23: String', s24: String', s25: String', s26: String', s27: String', s28: Boolean', s29: scala.math.BigDecimal', s30: String', s31: String', s32: Boolean', s33: Option[String'], s34: String', s35: Option[String'], s36: Option[String'], s37: java.sql.Date', s38: Option[String'], s39: Option[String'], s40: String', s41: Option[String'], s42: String', s43: Option[scala.math.BigDecimal'], s44: scala.math.BigDecimal', s45: String', s46: Option[String'], s47: SOption[OptionDisc/Int], s48: Option[String'], s49: Option[String'], s50: Option[String'], s51: Option[String'], s52: Option[String'], s53: Option[String'], s54: Option[Int'], s55: Option[String'], s56: Option[String'], s57: Option[scala.math.BigDecimal'], s58: Option[String'], s59: Option[String'], s60: Option[String'], s61: Option[String'], s62: Option[String'], s63: Option[Boolean'], s64: Option[scala.math.BigDecimal'], s65: Option[String'], s66: Option[String'], s67: Option[Boolean'], s68: SOption[OptionDisc/Int], s69: Option[String'], s70: Option[String'], s71: Option[String'], s72: Option[String'], s73: Option[String'], s74: Option[String'], s75: Option[Int'], s76: Option[String'], s77: Option[String'], s78: Option[scala.math.BigDecimal'], s79: Option[String'], s80: Option[String'], s81: Option[String'], s82: Option[String'], s83: Option[String'], s84: Option[Boolean'], s85: Option[scala.math.BigDecimal'], s86: Option[String'], s87: Option[String'], s88: Option[Boolean']}>]
        from s89: Join Left : Vector[(t92<{s93: String', s94: String', s95: String', s96: String', s97: String', s98: Option[String'], s99: Int', s100: Option[String'], s101: String', s102: scala.math.BigDecimal', s103: String', s104: String', s105: String', s106: String', s107: String', s108: Boolean', s109: scala.math.BigDecimal', s110: String', s111: String', s112: Boolean', s113: Option[String'], s114: String', s115: Option[String'], s116: Option[String'], s117: java.sql.Date', s118: Option[String'], s119: Option[String'], s120: String', s121: Option[String'], s122: String', s123: Option[scala.math.BigDecimal'], s124: scala.math.BigDecimal', s125: String', s126: Option[String'], s127: SOption[OptionDisc/Int], s128: Option[String'], s129: Option[String'], s130: Option[String'], s131: Option[String'], s132: Option[String'], s133: Option[String'], s134: Option[Int'], s135: Option[String'], s136: Option[String'], s137: Option[scala.math.BigDecimal'], s138: Option[String'], s139: Option[String'], s140: Option[String'], s141: Option[String'], s142: Option[String'], s143: Option[Boolean'], s144: Option[scala.math.BigDecimal'], s145: Option[String'], s146: Option[String'], s147: Option[Boolean']}>, t148<{s149: String', s150: String', s151: String', s152: String', s153: String', s154: String', s155: Option[String'], s156: Int', s157: Option[String'], s158: String', s159: scala.math.BigDecimal', s160: String', s161: String', s162: String', s163: String', s164: Boolean', s165: scala.math.BigDecimal', s166: String', s167: String', s168: Boolean'}>)]
          left s90: Bind : Vector[t92<{s93: String', s94: String', s95: String', s96: String', s97: String', s98: Option[String'], s99: Int', s100: Option[String'], s101: String', s102: scala.math.BigDecimal', s103: String', s104: String', s105: String', s106: String', s107: String', s108: Boolean', s109: scala.math.BigDecimal', s110: String', s111: String', s112: Boolean', s113: Option[String'], s114: String', s115: Option[String'], s116: Option[String'], s117: java.sql.Date', s118: Option[String'], s119: Option[String'], s120: String', s121: Option[String'], s122: String', s123: Option[scala.math.BigDecimal'], s124: scala.math.BigDecimal', s125: String', s126: Option[String'], s127: SOption[OptionDisc/Int], s128: Option[String'], s129: Option[String'], s130: Option[String'], s131: Option[String'], s132: Option[String'], s133: Option[String'], s134: Option[Int'], s135: Option[String'], s136: Option[String'], s137: Option[scala.math.BigDecimal'], s138: Option[String'], s139: Option[String'], s140: Option[String'], s141: Option[String'], s142: Option[String'], s143: Option[Boolean'], s144: Option[scala.math.BigDecimal'], s145: Option[String'], s146: Option[String'], s147: Option[Boolean']}>]
            from s169: Join Left : Vector[(t172<{s173: String', s174: String', s175: String', s176: String', s177: String', s178: Option[String'], s179: Int', s180: Option[String'], s181: String', s182: scala.math.BigDecimal', s183: String', s184: String', s185: String', s186: String', s187: String', s188: Boolean', s189: scala.math.BigDecimal', s190: String', s191: String', s192: Boolean', s193: Option[String'], s194: String', s195: Option[String'], s196: Option[String'], s197: java.sql.Date', s198: Option[String'], s199: Option[String'], s200: String', s201: Option[String'], s202: String', s203: Option[scala.math.BigDecimal'], s204: scala.math.BigDecimal', s205: String', s206: Option[String']}>, t207<{s208: String', s209: String', s210: String', s211: String', s212: String', s213: String', s214: Option[String'], s215: Int', s216: Option[String'], s217: String', s218: scala.math.BigDecimal', s219: String', s220: String', s221: String', s222: String', s223: Boolean', s224: scala.math.BigDecimal', s225: String', s226: String', s227: Boolean'}>)]
              left s170: Bind : Vector[t172<{s173: String', s174: String', s175: String', s176: String', s177: String', s178: Option[String'], s179: Int', s180: Option[String'], s181: String', s182: scala.math.BigDecimal', s183: String', s184: String', s185: String', s186: String', s187: String', s188: Boolean', s189: scala.math.BigDecimal', s190: String', s191: String', s192: Boolean', s193: Option[String'], s194: String', s195: Option[String'], s196: Option[String'], s197: java.sql.Date', s198: Option[String'], s199: Option[String'], s200: String', s201: Option[String'], s202: String', s203: Option[scala.math.BigDecimal'], s204: scala.math.BigDecimal', s205: String', s206: Option[String']}>]
                from s228: Join Inner : Vector[(t231<{s232: String', s233: String', s234: String', s235: String', s236: String', s237: Option[String'], s238: Int', s239: Option[String'], s240: String', s241: scala.math.BigDecimal', s242: String', s243: String', s244: String', s245: String', s246: String', s247: Boolean', s248: scala.math.BigDecimal', s249: String', s250: String', s251: Boolean'}>, t252<{s253: Option[String'], s254: String', s255: Option[String'], s256: Option[String'], s257: java.sql.Date', s258: Option[String'], s259: Option[String'], s260: String', s261: Option[String'], s262: String', s263: Option[scala.math.BigDecimal'], s264: scala.math.BigDecimal', s265: String', s266: Option[String']}>)]
                  left s229: Bind : Vector[t231<{s232: String', s233: String', s234: String', s235: String', s236: String', s237: Option[String'], s238: Int', s239: Option[String'], s240: String', s241: scala.math.BigDecimal', s242: String', s243: String', s244: String', s245: String', s246: String', s247: Boolean', s248: scala.math.BigDecimal', s249: String', s250: String', s251: Boolean'}>]
                    from REDACTED: Table REDACTED : Vector[@t268<{REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Option[String'], REDACTED: Int', REDACTED: Option[String'], REDACTED: String', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Boolean', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: Boolean'}>]
                    select: Pure t231 : Vector[t231<{s232: String', s233: String', s234: String', s235: String', s236: String', s237: Option[String'], s238: Int', s239: Option[String'], s240: String', s241: scala.math.BigDecimal', s242: String', s243: String', s244: String', s245: String', s246: String', s247: Boolean', s248: scala.math.BigDecimal', s249: String', s250: String', s251: Boolean'}>]
                      value: StructNode : {s232: String', s233: String', s234: String', s235: String', s236: String', s237: Option[String'], s238: Int', s239: Option[String'], s240: String', s241: scala.math.BigDecimal', s242: String', s243: String', s244: String', s245: String', s246: String', s247: Boolean', s248: scala.math.BigDecimal', s249: String', s250: String', s251: Boolean'}
                        s232: Path s267.REDACTED : String'
                        s233: Path s267.REDACTED : String'
                        s234: Path s267.REDACTED : String'
                        s235: Path s267.REDACTED : String'
                        s236: Path s267.REDACTED : String'
                        s237: Path s267.REDACTED : Option[String']
                        s238: Path s267.REDACTED : Int'
                        s239: Path s267.REDACTED : Option[String']
                        s240: Path s267.REDACTED : String'
                        s241: Path s267.REDACTED : scala.math.BigDecimal'
                        s242: Path s267.REDACTED : String'
                        s243: Path s267.REDACTED : String'
                        s244: Path s267.REDACTED : String'
                        s245: Path s267.REDACTED : String'
                        s246: Path s267.REDACTED : String'
                        s247: Path s267.REDACTED : Boolean'
                        s248: Path s267.REDACTED : scala.math.BigDecimal'
                        s249: Path s267.REDACTED : String'
                        s250: Path s267.REDACTED : String'
                        s251: Path s267.REDACTED : Boolean'
                  right s230: Bind : Vector[t252<{s253: Option[String'], s254: String', s255: Option[String'], s256: Option[String'], s257: java.sql.Date', s258: Option[String'], s259: Option[String'], s260: String', s261: Option[String'], s262: String', s263: Option[scala.math.BigDecimal'], s264: scala.math.BigDecimal', s265: String', s266: Option[String']}>]
                    from REDACTED: Table REDACTED : Vector[@t270<{REDACTED: Option[String'], REDACTED: String', REDACTED: Option[String'], REDACTED: Option[String'], REDACTED: java.sql.Date', REDACTED: Option[String'], REDACTED: Option[String'], REDACTED: String', REDACTED: Option[String'], REDACTED: String', REDACTED: Option[scala.math.BigDecimal'], REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: Option[String']}>]
                    select: Pure t252 : Vector[t252<{s253: Option[String'], s254: String', s255: Option[String'], s256: Option[String'], s257: java.sql.Date', s258: Option[String'], s259: Option[String'], s260: String', s261: Option[String'], s262: String', s263: Option[scala.math.BigDecimal'], s264: scala.math.BigDecimal', s265: String', s266: Option[String']}>]
                      value: StructNode : {s253: Option[String'], s254: String', s255: Option[String'], s256: Option[String'], s257: java.sql.Date', s258: Option[String'], s259: Option[String'], s260: String', s261: Option[String'], s262: String', s263: Option[scala.math.BigDecimal'], s264: scala.math.BigDecimal', s265: String', s266: Option[String']}
                        s253: Path s269.REDACTED : Option[String']
                        s254: Path s269.REDACTED : String'
                        s255: Path s269.REDACTED : Option[String']
                        s256: Path s269.REDACTED : Option[String']
                        s257: Path s269.REDACTED : java.sql.Date'
                        s258: Path s269.REDACTED : Option[String']
                        s259: Path s269.REDACTED : Option[String']
                        s260: Path s269.REDACTED : String'
                        s261: Path s269.REDACTED : Option[String']
                        s262: Path s269.REDACTED : String'
                        s263: Path s269.REDACTED : Option[scala.math.BigDecimal']
                        s264: Path s269.REDACTED : scala.math.BigDecimal'
                        s265: Path s269.REDACTED : String'
                        s266: Path s269.REDACTED : Option[String']
                  on: Apply Function = : Boolean
                    0: Path s229.REDACTED : String'
                    1: Path s230.REDACTED : String'
                select: Pure t172 : Vector[t172<{s173: String', s174: String', s175: String', s176: String', s177: String', s178: Option[String'], s179: Int', s180: Option[String'], s181: String', s182: scala.math.BigDecimal', s183: String', s184: String', s185: String', s186: String', s187: String', s188: Boolean', s189: scala.math.BigDecimal', s190: String', s191: String', s192: Boolean', s193: Option[String'], s194: String', s195: Option[String'], s196: Option[String'], s197: java.sql.Date', s198: Option[String'], s199: Option[String'], s200: String', s201: Option[String'], s202: String', s203: Option[scala.math.BigDecimal'], s204: scala.math.BigDecimal', s205: String', s206: Option[String']}>]
                  value: StructNode : {s173: String', s174: String', s175: String', s176: String', s177: String', s178: Option[String'], s179: Int', s180: Option[String'], s181: String', s182: scala.math.BigDecimal', s183: String', s184: String', s185: String', s186: String', s187: String', s188: Boolean', s189: scala.math.BigDecimal', s190: String', s191: String', s192: Boolean', s193: Option[String'], s194: String', s195: Option[String'], s196: Option[String'], s197: java.sql.Date', s198: Option[String'], s199: Option[String'], s200: String', s201: Option[String'], s202: String', s203: Option[scala.math.BigDecimal'], s204: scala.math.BigDecimal', s205: String', s206: Option[String']}
                    s173: Path s228._1.s232 : String'
                    s174: Path s228._1.s233 : String'
                    s175: Path s228._1.s234 : String'
                    s176: Path s228._1.s235 : String'
                    s177: Path s228._1.s236 : String'
                    s178: Path s228._1.s237 : Option[String']
                    s179: Path s228._1.s238 : Int'
                    s180: Path s228._1.s239 : Option[String']
                    s181: Path s228._1.s240 : String'
                    s182: Path s228._1.s241 : scala.math.BigDecimal'
                    s183: Path s228._1.s242 : String'
                    s184: Path s228._1.s243 : String'
                    s185: Path s228._1.s244 : String'
                    s186: Path s228._1.s245 : String'
                    s187: Path s228._1.s246 : String'
                    s188: Path s228._1.s247 : Boolean'
                    s189: Path s228._1.s248 : scala.math.BigDecimal'
                    s190: Path s228._1.s249 : String'
                    s191: Path s228._1.s250 : String'
                    s192: Path s228._1.s251 : Boolean'
                    s193: Path s228._2.s253 : Option[String']
                    s194: Path s228._2.s254 : String'
                    s195: Path s228._2.s255 : Option[String']
                    s196: Path s228._2.s256 : Option[String']
                    s197: Path s228._2.s257 : java.sql.Date'
                    s198: Path s228._2.s258 : Option[String']
                    s199: Path s228._2.s259 : Option[String']
                    s200: Path s228._2.s260 : String'
                    s201: Path s228._2.s261 : Option[String']
                    s202: Path s228._2.s262 : String'
                    s203: Path s228._2.s263 : Option[scala.math.BigDecimal']
                    s204: Path s228._2.s264 : scala.math.BigDecimal'
                    s205: Path s228._2.s265 : String'
                    s206: Path s228._2.s266 : Option[String']
              right s171: Bind : Vector[t207<{s208: String', s209: String', s210: String', s211: String', s212: String', s213: String', s214: Option[String'], s215: Int', s216: Option[String'], s217: String', s218: scala.math.BigDecimal', s219: String', s220: String', s221: String', s222: String', s223: Boolean', s224: scala.math.BigDecimal', s225: String', s226: String', s227: Boolean'}>]
                from REDACTED: Table REDACTED : Vector[@t272<{REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Option[String'], REDACTED: Int', REDACTED: Option[String'], REDACTED: String', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Boolean', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: Boolean'}>]
                select: Pure t207 : Vector[t207<{s208: String', s209: String', s210: String', s211: String', s212: String', s213: String', s214: Option[String'], s215: Int', s216: Option[String'], s217: String', s218: scala.math.BigDecimal', s219: String', s220: String', s221: String', s222: String', s223: Boolean', s224: scala.math.BigDecimal', s225: String', s226: String', s227: Boolean'}>]
                  value: StructNode : {s208: String', s209: String', s210: String', s211: String', s212: String', s213: String', s214: Option[String'], s215: Int', s216: Option[String'], s217: String', s218: scala.math.BigDecimal', s219: String', s220: String', s221: String', s222: String', s223: Boolean', s224: scala.math.BigDecimal', s225: String', s226: String', s227: Boolean'}
                    s208: Path s271.REDACTED : String'
                    s209: Path s271.REDACTED : String'
                    s210: Path s271.REDACTED : String'
                    s211: Path s271.REDACTED : String'
                    s212: Path s271.REDACTED : String'
                    s213: Path s271.REDACTED : String'
                    s214: Path s271.REDACTED : Option[String']
                    s215: Path s271.REDACTED : Int'
                    s216: Path s271.REDACTED : Option[String']
                    s217: Path s271.REDACTED : String'
                    s218: Path s271.REDACTED : scala.math.BigDecimal'
                    s219: Path s271.REDACTED : String'
                    s220: Path s271.REDACTED : String'
                    s221: Path s271.REDACTED : String'
                    s222: Path s271.REDACTED : String'
                    s223: Path s271.REDACTED : Boolean'
                    s224: Path s271.REDACTED : scala.math.BigDecimal'
                    s225: Path s271.REDACTED : String'
                    s226: Path s271.REDACTED : String'
                    s227: Path s271.REDACTED : Boolean'
              on: Apply Function = : SOption[Boolean]
                0: Path s170.REDACTED : Option[String']
                1: Path s171.REDACTED : String'
            select: Pure t92 : Vector[t92<{s93: String', s94: String', s95: String', s96: String', s97: String', s98: Option[String'], s99: Int', s100: Option[String'], s101: String', s102: scala.math.BigDecimal', s103: String', s104: String', s105: String', s106: String', s107: String', s108: Boolean', s109: scala.math.BigDecimal', s110: String', s111: String', s112: Boolean', s113: Option[String'], s114: String', s115: Option[String'], s116: Option[String'], s117: java.sql.Date', s118: Option[String'], s119: Option[String'], s120: String', s121: Option[String'], s122: String', s123: Option[scala.math.BigDecimal'], s124: scala.math.BigDecimal', s125: String', s126: Option[String'], s127: SOption[OptionDisc/Int], s128: Option[String'], s129: Option[String'], s130: Option[String'], s131: Option[String'], s132: Option[String'], s133: Option[String'], s134: Option[Int'], s135: Option[String'], s136: Option[String'], s137: Option[scala.math.BigDecimal'], s138: Option[String'], s139: Option[String'], s140: Option[String'], s141: Option[String'], s142: Option[String'], s143: Option[Boolean'], s144: Option[scala.math.BigDecimal'], s145: Option[String'], s146: Option[String'], s147: Option[Boolean']}>]
              value: StructNode : {s93: String', s94: String', s95: String', s96: String', s97: String', s98: Option[String'], s99: Int', s100: Option[String'], s101: String', s102: scala.math.BigDecimal', s103: String', s104: String', s105: String', s106: String', s107: String', s108: Boolean', s109: scala.math.BigDecimal', s110: String', s111: String', s112: Boolean', s113: Option[String'], s114: String', s115: Option[String'], s116: Option[String'], s117: java.sql.Date', s118: Option[String'], s119: Option[String'], s120: String', s121: Option[String'], s122: String', s123: Option[scala.math.BigDecimal'], s124: scala.math.BigDecimal', s125: String', s126: Option[String'], s127: SOption[OptionDisc/Int], s128: Option[String'], s129: Option[String'], s130: Option[String'], s131: Option[String'], s132: Option[String'], s133: Option[String'], s134: Option[Int'], s135: Option[String'], s136: Option[String'], s137: Option[scala.math.BigDecimal'], s138: Option[String'], s139: Option[String'], s140: Option[String'], s141: Option[String'], s142: Option[String'], s143: Option[Boolean'], s144: Option[scala.math.BigDecimal'], s145: Option[String'], s146: Option[String'], s147: Option[Boolean']}
                s93: Path s169._1.s173 : String'
                s94: Path s169._1.s174 : String'
                s95: Path s169._1.s175 : String'
                s96: Path s169._1.s176 : String'
                s97: Path s169._1.s177 : String'
                s98: Path s169._1.s178 : Option[String']
                s99: Path s169._1.s179 : Int'
                s100: Path s169._1.s180 : Option[String']
                s101: Path s169._1.s181 : String'
                s102: Path s169._1.s182 : scala.math.BigDecimal'
                s103: Path s169._1.s183 : String'
                s104: Path s169._1.s184 : String'
                s105: Path s169._1.s185 : String'
                s106: Path s169._1.s186 : String'
                s107: Path s169._1.s187 : String'
                s108: Path s169._1.s188 : Boolean'
                s109: Path s169._1.s189 : scala.math.BigDecimal'
                s110: Path s169._1.s190 : String'
                s111: Path s169._1.s191 : String'
                s112: Path s169._1.s192 : Boolean'
                s113: Path s169._1.s193 : Option[String']
                s114: Path s169._1.s194 : String'
                s115: Path s169._1.s195 : Option[String']
                s116: Path s169._1.s196 : Option[String']
                s117: Path s169._1.s197 : java.sql.Date'
                s118: Path s169._1.s198 : Option[String']
                s119: Path s169._1.s199 : Option[String']
                s120: Path s169._1.s200 : String'
                s121: Path s169._1.s201 : Option[String']
                s122: Path s169._1.s202 : String'
                s123: Path s169._1.s203 : Option[scala.math.BigDecimal']
                s124: Path s169._1.s204 : scala.math.BigDecimal'
                s125: Path s169._1.s205 : String'
                s126: Path s169._1.s206 : Option[String']
                s127: IfThenElse : SOption[OptionDisc/Int]
                  if: Apply Function = : Boolean
                    0: Apply Function SilentCast : Option[String']
                      0: Path s169._2.s208 : String'
                    1: LiteralNode null (volatileHint=false) : Null
                  then: LiteralNode None (volatileHint=false) : SOption[OptionDisc/Int]
                  else: LiteralNode Some(1) (volatileHint=false) : SOption[OptionDisc/Int]
                s128: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s209 : String'
                s129: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s210 : String'
                s130: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s211 : String'
                s131: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s212 : String'
                s132: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s213 : String'
                s133: Path s169._2.s214 : Option[String']
                s134: Apply Function SilentCast : Option[Int']
                  0: Path s169._2.s215 : Int'
                s135: Path s169._2.s216 : Option[String']
                s136: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s217 : String'
                s137: Apply Function SilentCast : Option[scala.math.BigDecimal']
                  0: Path s169._2.s218 : scala.math.BigDecimal'
                s138: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s219 : String'
                s139: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s208 : String'
                s140: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s220 : String'
                s141: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s221 : String'
                s142: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s222 : String'
                s143: Apply Function SilentCast : Option[Boolean']
                  0: Path s169._2.s223 : Boolean'
                s144: Apply Function SilentCast : Option[scala.math.BigDecimal']
                  0: Path s169._2.s224 : scala.math.BigDecimal'
                s145: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s225 : String'
                s146: Apply Function SilentCast : Option[String']
                  0: Path s169._2.s226 : String'
                s147: Apply Function SilentCast : Option[Boolean']
                  0: Path s169._2.s227 : Boolean'
          right s91: Bind : Vector[t148<{s149: String', s150: String', s151: String', s152: String', s153: String', s154: String', s155: Option[String'], s156: Int', s157: Option[String'], s158: String', s159: scala.math.BigDecimal', s160: String', s161: String', s162: String', s163: String', s164: Boolean', s165: scala.math.BigDecimal', s166: String', s167: String', s168: Boolean'}>]
            from REDACTED: Table REDACTED : Vector[@t274<{REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Option[String'], REDACTED: Int', REDACTED: Option[String'], REDACTED: String', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: String', REDACTED: Boolean', REDACTED: scala.math.BigDecimal', REDACTED: String', REDACTED: String', REDACTED: Boolean'}>]
            select: Pure t148 : Vector[t148<{s149: String', s150: String', s151: String', s152: String', s153: String', s154: String', s155: Option[String'], s156: Int', s157: Option[String'], s158: String', s159: scala.math.BigDecimal', s160: String', s161: String', s162: String', s163: String', s164: Boolean', s165: scala.math.BigDecimal', s166: String', s167: String', s168: Boolean'}>]
              value: StructNode : {s149: String', s150: String', s151: String', s152: String', s153: String', s154: String', s155: Option[String'], s156: Int', s157: Option[String'], s158: String', s159: scala.math.BigDecimal', s160: String', s161: String', s162: String', s163: String', s164: Boolean', s165: scala.math.BigDecimal', s166: String', s167: String', s168: Boolean'}
                s149: Path s273.REDACTED : String'
                s150: Path s273.REDACTED : String'
                s151: Path s273.REDACTED : String'
                s152: Path s273.REDACTED : String'
                s153: Path s273.REDACTED : String'
                s154: Path s273.REDACTED : String'
                s155: Path s273.REDACTED : Option[String']
                s156: Path s273.REDACTED : Int'
                s157: Path s273.REDACTED : Option[String']
                s158: Path s273.REDACTED : String'
                s159: Path s273.REDACTED : scala.math.BigDecimal'
                s160: Path s273.REDACTED : String'
                s161: Path s273.REDACTED : String'
                s162: Path s273.REDACTED : String'
                s163: Path s273.REDACTED : String'
                s164: Path s273.REDACTED : Boolean'
                s165: Path s273.REDACTED : scala.math.BigDecimal'
                s166: Path s273.REDACTED : String'
                s167: Path s273.REDACTED : String'
                s168: Path s273.REDACTED : Boolean'
          on: Apply Function = : SOption[Boolean]
            0: Path s90.REDACTED : Option[String']
            1: Path s91.REDACTED : String'
        select: Pure t12 : Vector[t12<{s13: String', s14: String', s15: String', s16: String', s17: String', s18: Option[String'], s19: Int', s20: Option[String'], s21: String', s22: scala.math.BigDecimal', s23: String', s24: String', s25: String', s26: String', s27: String', s28: Boolean', s29: scala.math.BigDecimal', s30: String', s31: String', s32: Boolean', s33: Option[String'], s34: String', s35: Option[String'], s36: Option[String'], s37: java.sql.Date', s38: Option[String'], s39: Option[String'], s40: String', s41: Option[String'], s42: String', s43: Option[scala.math.BigDecimal'], s44: scala.math.BigDecimal', s45: String', s46: Option[String'], s47: SOption[OptionDisc/Int], s48: Option[String'], s49: Option[String'], s50: Option[String'], s51: Option[String'], s52: Option[String'], s53: Option[String'], s54: Option[Int'], s55: Option[String'], s56: Option[String'], s57: Option[scala.math.BigDecimal'], s58: Option[String'], s59: Option[String'], s60: Option[String'], s61: Option[String'], s62: Option[String'], s63: Option[Boolean'], s64: Option[scala.math.BigDecimal'], s65: Option[String'], s66: Option[String'], s67: Option[Boolean'], s68: SOption[OptionDisc/Int], s69: Option[String'], s70: Option[String'], s71: Option[String'], s72: Option[String'], s73: Option[String'], s74: Option[String'], s75: Option[Int'], s76: Option[String'], s77: Option[String'], s78: Option[scala.math.BigDecimal'], s79: Option[String'], s80: Option[String'], s81: Option[String'], s82: Option[String'], s83: Option[String'], s84: Option[Boolean'], s85: Option[scala.math.BigDecimal'], s86: Option[String'], s87: Option[String'], s88: Option[Boolean']}>]
          value: StructNode : {s13: String', s14: String', s15: String', s16: String', s17: String', s18: Option[String'], s19: Int', s20: Option[String'], s21: String', s22: scala.math.BigDecimal', s23: String', s24: String', s25: String', s26: String', s27: String', s28: Boolean', s29: scala.math.BigDecimal', s30: String', s31: String', s32: Boolean', s33: Option[String'], s34: String', s35: Option[String'], s36: Option[String'], s37: java.sql.Date', s38: Option[String'], s39: Option[String'], s40: String', s41: Option[String'], s42: String', s43: Option[scala.math.BigDecimal'], s44: scala.math.BigDecimal', s45: String', s46: Option[String'], s47: SOption[OptionDisc/Int], s48: Option[String'], s49: Option[String'], s50: Option[String'], s51: Option[String'], s52: Option[String'], s53: Option[String'], s54: Option[Int'], s55: Option[String'], s56: Option[String'], s57: Option[scala.math.BigDecimal'], s58: Option[String'], s59: Option[String'], s60: Option[String'], s61: Option[String'], s62: Option[String'], s63: Option[Boolean'], s64: Option[scala.math.BigDecimal'], s65: Option[String'], s66: Option[String'], s67: Option[Boolean'], s68: SOption[OptionDisc/Int], s69: Option[String'], s70: Option[String'], s71: Option[String'], s72: Option[String'], s73: Option[String'], s74: Option[String'], s75: Option[Int'], s76: Option[String'], s77: Option[String'], s78: Option[scala.math.BigDecimal'], s79: Option[String'], s80: Option[String'], s81: Option[String'], s82: Option[String'], s83: Option[String'], s84: Option[Boolean'], s85: Option[scala.math.BigDecimal'], s86: Option[String'], s87: Option[String'], s88: Option[Boolean']}
            s13: Path s89._1.s93 : String'
            s14: Path s89._1.s94 : String'
            s15: Path s89._1.s95 : String'
            s16: Path s89._1.s96 : String'
            s17: Path s89._1.s97 : String'
            s18: Path s89._1.s98 : Option[String']
            s19: Path s89._1.s99 : Int'
            s20: Path s89._1.s100 : Option[String']
            s21: Path s89._1.s101 : String'
            s22: Path s89._1.s102 : scala.math.BigDecimal'
            s23: Path s89._1.s103 : String'
            s24: Path s89._1.s104 : String'
            s25: Path s89._1.s105 : String'
            s26: Path s89._1.s106 : String'
            s27: Path s89._1.s107 : String'
            s28: Path s89._1.s108 : Boolean'
            s29: Path s89._1.s109 : scala.math.BigDecimal'
            s30: Path s89._1.s110 : String'
            s31: Path s89._1.s111 : String'
            s32: Path s89._1.s112 : Boolean'
            s33: Path s89._1.s113 : Option[String']
            s34: Path s89._1.s114 : String'
            s35: Path s89._1.s115 : Option[String']
            s36: Path s89._1.s116 : Option[String']
            s37: Path s89._1.s117 : java.sql.Date'
            s38: Path s89._1.s118 : Option[String']
            s39: Path s89._1.s119 : Option[String']
            s40: Path s89._1.s120 : String'
            s41: Path s89._1.s121 : Option[String']
            s42: Path s89._1.s122 : String'
            s43: Path s89._1.s123 : Option[scala.math.BigDecimal']
            s44: Path s89._1.s124 : scala.math.BigDecimal'
            s45: Path s89._1.s125 : String'
            s46: Path s89._1.s126 : Option[String']
            s47: Path s89._1.s127 : SOption[OptionDisc/Int]
            s48: Path s89._1.s128 : Option[String']
            s49: Path s89._1.s129 : Option[String']
            s50: Path s89._1.s130 : Option[String']
            s51: Path s89._1.s131 : Option[String']
            s52: Path s89._1.s132 : Option[String']
            s53: Path s89._1.s133 : Option[String']
            s54: Path s89._1.s134 : Option[Int']
            s55: Path s89._1.s135 : Option[String']
            s56: Path s89._1.s136 : Option[String']
            s57: Path s89._1.s137 : Option[scala.math.BigDecimal']
            s58: Path s89._1.s138 : Option[String']
            s59: Path s89._1.s139 : Option[String']
            s60: Path s89._1.s140 : Option[String']
            s61: Path s89._1.s141 : Option[String']
            s62: Path s89._1.s142 : Option[String']
            s63: Path s89._1.s143 : Option[Boolean']
            s64: Path s89._1.s144 : Option[scala.math.BigDecimal']
            s65: Path s89._1.s145 : Option[String']
            s66: Path s89._1.s146 : Option[String']
            s67: Path s89._1.s147 : Option[Boolean']
            s68: IfThenElse : SOption[OptionDisc/Int]
              if: Apply Function = : Boolean
                0: Apply Function SilentCast : Option[String']
                  0: Path s89._2.s149 : String'
                1: LiteralNode null (volatileHint=false) : Null
              then: LiteralNode None (volatileHint=false) : SOption[OptionDisc/Int]
              else: LiteralNode Some(1) (volatileHint=false) : SOption[OptionDisc/Int]
            s69: Apply Function SilentCast : Option[String']
              0: Path s89._2.s150 : String'
            s70: Apply Function SilentCast : Option[String']
              0: Path s89._2.s151 : String'
            s71: Apply Function SilentCast : Option[String']
              0: Path s89._2.s152 : String'
            s72: Apply Function SilentCast : Option[String']
              0: Path s89._2.s153 : String'
            s73: Apply Function SilentCast : Option[String']
              0: Path s89._2.s154 : String'
            s74: Path s89._2.s155 : Option[String']
            s75: Apply Function SilentCast : Option[Int']
              0: Path s89._2.s156 : Int'
            s76: Path s89._2.s157 : Option[String']
            s77: Apply Function SilentCast : Option[String']
              0: Path s89._2.s158 : String'
            s78: Apply Function SilentCast : Option[scala.math.BigDecimal']
              0: Path s89._2.s159 : scala.math.BigDecimal'
            s79: Apply Function SilentCast : Option[String']
              0: Path s89._2.s160 : String'
            s80: Apply Function SilentCast : Option[String']
              0: Path s89._2.s149 : String'
            s81: Apply Function SilentCast : Option[String']
              0: Path s89._2.s161 : String'
            s82: Apply Function SilentCast : Option[String']
              0: Path s89._2.s162 : String'
            s83: Apply Function SilentCast : Option[String']
              0: Path s89._2.s163 : String'
            s84: Apply Function SilentCast : Option[Boolean']
              0: Path s89._2.s164 : Boolean'
            s85: Apply Function SilentCast : Option[scala.math.BigDecimal']
              0: Path s89._2.s165 : scala.math.BigDecimal'
            s86: Apply Function SilentCast : Option[String']
              0: Path s89._2.s166 : String'
            s87: Apply Function SilentCast : Option[String']
              0: Path s89._2.s167 : String'
            s88: Apply Function SilentCast : Option[Boolean']
              0: Path s89._2.s168 : Boolean'
      where: Apply Function and : Boolean
        0: Apply Function = : Boolean
          0: Path s11.REDACTED : String'
          1: QueryParameter s275 <function1> : String'
        1: Apply Function = : Boolean
          0: Path s11.REDACTED : String'
          1: QueryParameter s276 <function1> : String'
    select: Pure t4 : Vector[t4<{s5: Option[scala.math.BigDecimal'], s6: SOption[OptionDisc/Int], s7: Option[String'], s8: SOption[OptionDisc/Int], s9: Option[String']}>]
      value: StructNode : {s5: Option[scala.math.BigDecimal'], s6: SOption[OptionDisc/Int], s7: Option[String'], s8: SOption[OptionDisc/Int], s9: Option[String']}
        s5: Path s10.REDACTED : Option[scala.math.BigDecimal']
        s6: Path s10.REDACTED : SOption[OptionDisc/Int]
        s7: Path s10.REDACTED : Option[String']
        s8: Path s10.REDACTED : SOption[OptionDisc/Int]
        s9: Path s10.REDACTED : Option[String']
  by: ProductNode : (Option[String'], Option[String'])
    1: IfThenElse : Option[String']
      if: Apply Function not : Boolean
        0: Apply Function = : Boolean
          0: Path s2.REDACTED : SOption[OptionDisc/Int]
          1: LiteralNode null (volatileHint=false) : Null
      then: OptionApply : Option[String']
        0: Apply Function SilentCast : String'
          0: Path s2.REDACTED : Option[String']
      else: LiteralNode None (volatileHint=false) : Option[String']
    2: IfThenElse : Option[String']
      if: Apply Function not : Boolean
        0: Apply Function = : Boolean
          0: Path s2.REDACTED : SOption[OptionDisc/Int]
          1: LiteralNode null (volatileHint=false) : Null
      then: OptionApply : Option[String']
        0: Apply Function SilentCast : String'
          0: Path s2.REDACTED : Option[String']
      else: LiteralNode None (volatileHint=false) : Option[String']
  at slick.compiler.MergeToComprehensions$$anonfun$26.apply(MergeToComprehensions.scala:179)
  at slick.compiler.MergeToComprehensions$$anonfun$26.apply(MergeToComprehensions.scala:179)
  at scala.Option.getOrElse(Option.scala:121)
  at slick.compiler.MergeToComprehensions.slick$compiler$MergeToComprehensions$$convertBase$1(MergeToComprehensions.scala:179)
  at slick.compiler.MergeToComprehensions$$anonfun$slick$compiler$MergeToComprehensions$$mergeFilterWhere$1$2.apply(MergeToComprehensions.scala:173)
  at slick.compiler.MergeToComprehensions$$anonfun$slick$compiler$MergeToComprehensions$$mergeFilterWhere$1$2.apply(MergeToComprehensions.scala:173)
  at slick.compiler.MergeToComprehensions.mergeCommon(MergeToComprehensions.scala:356)
  at slick.compiler.MergeToComprehensions.slick$compiler$MergeToComprehensions$$mergeFilterWhere$1(MergeToComprehensions.scala:173)
  at slick.compiler.MergeToComprehensions.slick$compiler$MergeToComprehensions$$mergeGroupBy$1(MergeToComprehensions.scala:168)
  at slick.compiler.MergeToComprehensions$$anonfun$slick$compiler$MergeToComprehensions$$mergeSortBy$1$14.apply(MergeToComprehensions.scala:103)
  ...
```

<img style="margin: 0 auto; display: block" src="/img/doesnt-look-like-anything-to-me.jpg" alt="Doesn't look like anything to me">

Thanks for scrolling this far.

This compilation error was generated by a Scala Slick query that's essentially a SQL `SELECT` with a few `JOIN`s and a `GROUP BY`.
Try scrolling to the right for more laughs.

It took me a while to figure out, but the cause of the error was neglecting to call `.map` immediately after `.groupBy` to realize the resulting RHS `Query` to another completely inscrutable Slick type.

## Where did I start? 

Typically when I want to answer some question with a relational database, I start with a SQL query.
The specifics of this example aren't important, just the overall structure — a `SELECT` with three `JOIN`s, a `WHERE`, `GROUP BY` and `ORDER BY`:
```sql
SELECT foo.id AS foo_id, blah.id AS blah_id
FROM yep p
JOIN another_table c ON c.wat = p.wat
LEFT JOIN yep foo ON foo.huh = c.yah
LEFT JOIN yep bar ON bar.huh = c.nah
WHERE p.id = ? AND c.bar = ?
GROUP BY foo.id, blah.id
ORDER BY SUM(c.number) DESC;
```

Here's the equivalent Scala Slick query for that SQL query:
```scala
(fooId: Rep[String], someCode: Rep[String]) =>
  FooTable
    .join(BarTable)
    .on(_.a_id === _.a_id)
    .joinLeft(FooTable)
    .on {
      case ((_, bar), fooQux) =>
        bar.fooId === fooQux.fooId
    }
    .joinLeft(FooTable)
    .on {
      case (((_, bar), _), fooShazam) =>
        bar.fooId === fooShazam.fooId
    }
    .filter {
      case (((foo, bar), _), _) => foo.id === fooId && bar.someCode === someCode
    }
    .groupBy {
      case ((_, fooQux), fooShazam) =>
        (fooQux.map(_.id), fooShazam.map(_.id))
    }
    .map {
      case ((fooQuxId, genericQuxId), query) =>
        (fooQuxId, genericQuxId, query.map {
          case (((_, bar), _), _) => bar.number
        }.sum)
    }
    .sortBy { case (_, _, numberSum) => numberSum.desc }
    .map { case (fooQuxId, shazamQuxId, _) => (fooQuxId, shazamQuxId) }
```
I say it's equivalent but honestly I don't have the time or will to verify it.
It returns the same results given the same inputs, but I imagine the generated SQL is something much more baroque than the original SQL query.

This is supposed to be an improvement over writing SQL.

## ORMs make it _easy_

Is the Slick ORM query any easier to read than the SQL? It certainly took more time and effort to translate the original SQL query to Slick than it did to write the original SQL query.

It's type-safe, but to what benefit and at what cost? Note the ornate `case` tuple destructuring required in each intermediate Slick operation.
Also note that even though we know the exact _type_ of each table in each intermediate operation,
we've `JOIN`ed the same table twice and the only way to distinguish between them is to understand the way in which Slick embeds `JOIN`ed tables in increasingly-nested tuples — as far as I can tell it's impossible to uniquely name/alias the `JOIN`ed tables in this type system.

The only practical way to craft Slick queries is with additional tooling (IntelliJ + Scala plugin), and the whole time I was debugging this  IntelliJ said the query type-checked just fine.
I can't fault IntelliJ or the Scala plugin for giving up here. The type definitions underlying the Slick implementation are downright gymnastic.

Another surprising/frightening/amusing thing I've discovered with Slick: if you have a table with more than a couple dozen columns, well that's not really supported; see this [Stack Overflow](https://stackoverflow.com/questions/36618280/slick-codegen-tables-with-22-columns) Q&A for more on that. Apparently this issue has been _partially_ addressed with a recent release, but I'm pretty sure it's been supported in SQL for a long time now...

_I don't mean to beat up on just Slick here, even though it's freshest in my mind. I've experienced similar difficulties with other ORMs._

## Conclusion

The best DSL for SQL is SQL. Write SQL. Be free.
