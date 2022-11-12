## import

- xml, java 같은 파일들을 배열로 만들어 컴파일


``` java
final class Import extends TopLevelElement {

    private Stylesheet _imported = null;

    public Stylesheet getImportedStylesheet() {
        return _imported;
    }

    public void parseContents(final Parser parser) {
        final XSLTC xsltc = parser.getXSLTC();
        final Stylesheet context = parser.getCurrentStylesheet();

        try {
            String docToLoad = getAttribute("href");
            if (context.checkForLoop(docToLoad)) {
                final ErrorMsg msg = new ErrorMsg(ErrorMsg.CIRCULAR_INCLUDE_ERR,
                        docToLoad, this);
                parser.reportError(Constants.FATAL, msg);
                return;
            }

            InputSource input = null;
            XMLReader reader = null;
            String currLoadedDoc = context.getSystemId();
            SourceLoader loader = context.getSourceLoader();

            // Use SourceLoader if available
            if (loader != null) {
                input = loader.loadSource(docToLoad, currLoadedDoc, xsltc);
                if (input != null) {
                    docToLoad = input.getSystemId();
                    reader = xsltc.getXMLReader();
                } else if (parser.errorsFound()) {
                    return;
                }
            }

            // No SourceLoader or not resolved by SourceLoader
            if (input == null) {
                docToLoad = SystemIDResolver.getAbsoluteURI(docToLoad, currLoadedDoc);
                String accessError = SecuritySupport.checkAccess(docToLoad,
                        (String) xsltc.getProperty(XMLConstants.ACCESS_EXTERNAL_STYLESHEET),
                        XalanConstants.ACCESS_EXTERNAL_ALL);

                if (accessError != null) {
                    final ErrorMsg msg = new ErrorMsg(ErrorMsg.ACCESSING_XSLT_TARGET_ERR,
                            SecuritySupport.sanitizePath(docToLoad), accessError,
                            this);
                    parser.reportError(Constants.FATAL, msg);
                    return;
                }
                input = new InputSource(docToLoad);
            }

            // Return if we could not resolve the URL
            if (input == null) {
                final ErrorMsg msg = new ErrorMsg(ErrorMsg.FILE_NOT_FOUND_ERR, docToLoad, this);
                parser.reportError(Constants.FATAL, msg);
                return;
            }

            final SyntaxTreeNode root;
            if (reader != null) {
                root = parser.parse(reader, input);
            } else {
                root = parser.parse(input);
            }

            if (root == null)
                return;
            _imported = parser.makeStylesheet(root);
            if (_imported == null)
                return;

            _imported.setSourceLoader(loader);
            _imported.setSystemId(docToLoad);
            _imported.setParentStylesheet(context);
            _imported.setImportingStylesheet(context);
            _imported.setTemplateInlining(context.getTemplateInlining());

            // precedence for the including stylesheet
            final int currPrecedence = parser.getCurrentImportPrecedence();
            final int nextPrecedence = parser.getNextImportPrecedence();
            _imported.setImportPrecedence(currPrecedence);
            context.setImportPrecedence(nextPrecedence);
            parser.setCurrentStylesheet(_imported);
            _imported.parseContents(parser);

            final Iterator<SyntaxTreeNode> elements = _imported.elements();
            final Stylesheet topStylesheet = parser.getTopLevelStylesheet();
            while (elements.hasNext()) {
                final SyntaxTreeNode element = elements.next();
                if (element instanceof TopLevelElement) {
                    if (element instanceof Variable) {
                        topStylesheet.addVariable((Variable) element);
                    } else if (element instanceof Param) {
                        topStylesheet.addParam((Param) element);
                    } else {
                        topStylesheet.addElement((TopLevelElement) element);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            parser.setCurrentStylesheet(context);
        }
    }

    public Type typeCheck(SymbolTable stable) throws TypeCheckError {
        return Type.Void;
    }

    public void translate(ClassGenerator classGen, MethodGenerator methodGen) {
        // do nothing
    }
}


class TopLevelElement extends SyntaxTreeNode {

    /*
     * List of dependencies with other variables, parameters or
     * keys defined at the top level.
     */
    protected Vector _dependencies = null;

    /**
     * Type check all the children of this node.
     */
    public Type typeCheck(SymbolTable stable) throws TypeCheckError {
        return typeCheckContents(stable);
    }

    /**
     * Translate this node into JVM bytecodes.
     */
    public void translate(ClassGenerator classGen, MethodGenerator methodGen) {
        ErrorMsg msg = new ErrorMsg(ErrorMsg.NOT_IMPLEMENTED_ERR,
                                    getClass(), this);
        getParser().reportError(FATAL, msg);
    }

    /**
     * Translate this node into a fresh instruction list.
     * The original instruction list is saved and restored.
     */
    public InstructionList compile(ClassGenerator classGen,
                                   MethodGenerator methodGen) {
        final InstructionList result, save = methodGen.getInstructionList();
        methodGen.setInstructionList(result = new InstructionList());
        translate(classGen, methodGen);
        methodGen.setInstructionList(save);
        return result;
    }

    public void display(int indent) {
        indent(indent);
        Util.println("TopLevelElement");
        displayContents(indent + IndentIncrement);
    }

    /**
     * Add a dependency with other top-level elements like
     * variables, parameters or keys.
     */
    public void addDependency(TopLevelElement other) {
        if (_dependencies == null) {
            _dependencies = new Vector();
        }
        if (!_dependencies.contains(other)) {
            _dependencies.addElement(other);
        }
    }

    /**
     * Get the list of dependencies with other top-level elements
     * like variables, parameteres or keys.
     */
    public Vector getDependencies() {
        return _dependencies;
    }

}


public abstract class SyntaxTreeNode implements Constants {

    // Reference to the AST parser
    private Parser _parser;

    // AST navigation pointers
    protected SyntaxTreeNode _parent;          // Parent node
    private Stylesheet       _stylesheet;      // Stylesheet ancestor node
    private Template         _template;        // Template ancestor node
    private final List<SyntaxTreeNode> _contents = new ArrayList<>(2); // Child nodes

    // Element description data
    protected QName _qname;                    // The element QName
    private int _line;                         // Source file line number
    protected AttributesImpl _attributes = null;   // Attributes of this element
    private Map<String, String> _prefixMapping = null; // Namespace declarations

    // Sentinel - used to denote unrecognised syntaxt tree nodes.
    protected static final SyntaxTreeNode Dummy = new AbsolutePathPattern(null);

    // These two are used for indenting nodes in the AST (debug output)
    protected static final int IndentIncrement = 4;
    private static final char[] _spaces =
        "                                                       ".toCharArray();

    /**
     * Creates a new SyntaxTreeNode with a 'null' QName and no source file
     * line number reference.
     */
    public SyntaxTreeNode() {
        _line = 0;
        _qname = null;
    }

    /**
     * Creates a new SyntaxTreeNode with a 'null' QName.
     * @param line Source file line number reference
     */
    public SyntaxTreeNode(int line) {
        _line = line;
        _qname = null;
    }

    /**
     * Creates a new SyntaxTreeNode with no source file line number reference.
     * @param uri The element's namespace URI
     * @param prefix The element's namespace prefix
     * @param local The element's local name
     */
    public SyntaxTreeNode(String uri, String prefix, String local) {
        _line = 0;
        setQName(uri, prefix, local);
    }

    /**
     * Set the source file line number for this element
     * @param line The source file line number.
     */
    protected final void setLineNumber(int line) {
        _line = line;
    }

    /**
     * Get the source file line number for this element. If unavailable, lookup
     * in ancestors.
     *
     * @return The source file line number.
     */
    public final int getLineNumber() {
        if (_line > 0) return _line;
        SyntaxTreeNode parent = getParent();
        return (parent != null) ? parent.getLineNumber() : 0;
    }

    /**
     * Set the QName for the syntax tree node.
     * @param qname The QName for the syntax tree node
     */
    protected void setQName(QName qname) {
        _qname = qname;
    }

    /**
     * Set the QName for the SyntaxTreeNode
     * @param uri The element's namespace URI
     * @param prefix The element's namespace prefix
     * @param local The element's local name
     */
    protected void setQName(String uri, String prefix, String localname) {
        _qname = new QName(uri, prefix, localname);
    }

    /**
     * Set the QName for the SyntaxTreeNode
     * @param qname The QName for the syntax tree node
     */
    protected QName getQName() {
        return(_qname);
    }

    /**
     * Set the attributes for this SyntaxTreeNode.
     * @param attributes Attributes for the element. Must be passed in as an
     *                   implementation of org.xml.sax.Attributes.
     */
    protected void setAttributes(AttributesImpl attributes) {
        _attributes = attributes;
    }

    /**
     * Returns a value for an attribute from the source element.
     * @param qname The QName of the attribute to return.
     * @return The value of the attribute of name 'qname'.
     */
    protected String getAttribute(String qname) {
        if (_attributes == null) {
            return EMPTYSTRING;
        }
        final String value = _attributes.getValue(qname);
        return (value == null || value.equals(EMPTYSTRING)) ?
            EMPTYSTRING : value;
    }

    protected String getAttribute(String prefix, String localName) {
        return getAttribute(prefix + ':' + localName);
    }

    protected boolean hasAttribute(String qname) {
        return (_attributes != null && _attributes.getValue(qname) != null);
    }

    protected void addAttribute(String qname, String value) {
        int index = _attributes.getIndex(qname);
        if (index != -1) {
            _attributes.setAttribute(index, "", Util.getLocalName(qname),
                    qname, "CDATA", value);
        }
        else {
            _attributes.addAttribute("", Util.getLocalName(qname), qname,
                    "CDATA", value);
        }
    }

    /**
     * Returns a list of all attributes declared for the element represented by
     * this syntax tree node.
     * @return Attributes for this syntax tree node
     */
    protected Attributes getAttributes() {
        return(_attributes);
    }

    /**
     * Sets the prefix mapping for the namespaces that were declared in this
     * element. This does not include all prefix mappings in scope, so one
     * may have to check ancestor elements to get all mappings that are in
     * in scope. The prefixes must be passed in as a Map that maps
     * namespace prefixes (String objects) to namespace URIs (also String).
     * @param mapping The Map containing the mappings.
     */
    protected void setPrefixMapping(Map<String, String> mapping) {
        _prefixMapping = mapping;
    }

    /**
     * Returns a Map containing the prefix mappings that were declared
     * for this element. This does not include all prefix mappings in scope,
     * so one may have to check ancestor elements to get all mappings that are
     * in in scope.
     * @return Prefix mappings (for this element only).
     */
    protected Map<String, String> getPrefixMapping() {
        return _prefixMapping;
    }

    /**
     * Adds a single prefix mapping to this syntax tree node.
     * @param prefix Namespace prefix.
     * @param uri Namespace URI.
     */
    protected void addPrefixMapping(String prefix, String uri) {
        if (_prefixMapping == null)
            _prefixMapping = new HashMap<>();
        _prefixMapping.put(prefix, uri);
    }

    /**
     * Returns any namespace URI that is in scope for a given prefix. This
     * method checks namespace mappings for this element, and if necessary
     * for ancestor elements as well (ie. if the prefix maps to an URI in this
     * scope then you'll definately get the URI from this method).
     * @param prefix Namespace prefix.
     * @return Namespace URI.
     */
    protected String lookupNamespace(String prefix) {
        // Initialise the output (default is 'null' for undefined)
        String uri = null;

        // First look up the prefix/uri mapping in our own map...
        if (_prefixMapping != null)
            uri = _prefixMapping.get(prefix);
        // ... but if we can't find it there we ask our parent for the mapping
        if ((uri == null) && (_parent != null)) {
            uri = _parent.lookupNamespace(prefix);
            if ((prefix == Constants.EMPTYSTRING) && (uri == null))
                uri = Constants.EMPTYSTRING;
        }
        // ... and then we return whatever URI we've got.
        return(uri);
    }

    /**
     * Returns any namespace prefix that is mapped to a prefix in the current
     * scope. This method checks namespace mappings for this element, and if
     * necessary for ancestor elements as well (ie. if the URI is declared
     * within the current scope then you'll definately get the prefix from
     * this method). Note that this is a very slow method and consequentially
     * it should only be used strictly when needed.
     * @param uri Namespace URI.
     * @return Namespace prefix.
     */
    protected String lookupPrefix(String uri) {
        // Initialise the output (default is 'null' for undefined)
        String prefix = null;

        // First look up the prefix/uri mapping in our own map...
        if ((_prefixMapping != null) &&
            (_prefixMapping.containsValue(uri))) {
            for (Map.Entry<String, String> entry : _prefixMapping.entrySet()) {
                prefix = entry.getKey();
                String mapsTo = entry.getValue();
                if (mapsTo.equals(uri)) return(prefix);
            }
        }
        // ... but if we can't find it there we ask our parent for the mapping
        else if (_parent != null) {
            prefix = _parent.lookupPrefix(uri);
            if ((uri == Constants.EMPTYSTRING) && (prefix == null))
                prefix = Constants.EMPTYSTRING;
        }
        return(prefix);
    }

    /**
     * Set this node's parser. The parser (the XSLT parser) gives this
     * syntax tree node access to the symbol table and XPath parser.
     * @param parser The XSLT parser.
     */
    protected void setParser(Parser parser) {
        _parser = parser;
    }

    /**
     * Returns this node's XSLT parser.
     * @return The XSLT parser.
     */
    public final Parser getParser() {
        return _parser;
    }

    /**
     * Set this syntax tree node's parent node, if unset. For
     * re-parenting just use <code>node._parent = newparent</code>.
     *
     * @param parent The parent node.
     */
    protected void setParent(SyntaxTreeNode parent) {
        if (_parent == null) _parent = parent;
    }

    /**
     * Returns this syntax tree node's parent node.
     * @return The parent syntax tree node.
     */
    protected final SyntaxTreeNode getParent() {
        return _parent;
    }

    /**
     * Returns 'true' if this syntax tree node is the Sentinal node.
     * @return 'true' if this syntax tree node is the Sentinal node.
     */
    protected final boolean isDummy() {
        return this == Dummy;
    }

    /**
     * Get the import precedence of this element. The import precedence equals
     * the import precedence of the stylesheet in which this element occured.
     * @return The import precedence of this syntax tree node.
     */
    protected int getImportPrecedence() {
        Stylesheet stylesheet = getStylesheet();
        if (stylesheet == null) return Integer.MIN_VALUE;
        return stylesheet.getImportPrecedence();
    }

    /**
     * Get the Stylesheet node that represents the <xsl:stylesheet/> element
     * that this node occured under.
     * @return The Stylesheet ancestor node of this node.
     */
    public Stylesheet getStylesheet() {
        if (_stylesheet == null) {
            SyntaxTreeNode parent = this;
            while (parent != null) {
                if (parent instanceof Stylesheet)
                    return((Stylesheet)parent);
                parent = parent.getParent();
            }
            _stylesheet = (Stylesheet)parent;
        }
        return(_stylesheet);
    }

    /**
     * Get the Template node that represents the <xsl:template/> element
     * that this node occured under. Note that this method will return 'null'
     * for nodes that represent top-level elements.
     * @return The Template ancestor node of this node or 'null'.
     */
    protected Template getTemplate() {
        if (_template == null) {
            SyntaxTreeNode parent = this;
            while ((parent != null) && (!(parent instanceof Template)))
                parent = parent.getParent();
            _template = (Template)parent;
        }
        return(_template);
    }

    /**
     * Returns a reference to the XSLTC (XSLT compiler) in use.
     * @return XSLTC - XSLT compiler.
     */
    protected final XSLTC getXSLTC() {
        return _parser.getXSLTC();
    }

    /**
     * Returns the XSLT parser's symbol table.
     * @return Symbol table.
     */
    protected final SymbolTable getSymbolTable() {
        return (_parser == null) ? null : _parser.getSymbolTable();
    }

    /**
     * Parse the contents of this syntax tree nodes (child nodes, XPath
     * expressions, patterns and functions). The default behaviour is to parser
     * the syntax tree node's children (since there are no common expressions,
     * patterns, etc. that can be handled in this base class.
     * @param parser reference to the XSLT parser
     */
    public void parseContents(Parser parser) {
        parseChildren(parser);
    }

    /**
     * Parse all children of this syntax tree node. This method is normally
     * called by the parseContents() method.
     * @param parser reference to the XSLT parser
     */
    protected final void parseChildren(Parser parser) {

        List<QName> locals = null;   // only create when needed

        for (SyntaxTreeNode child : _contents) {
            parser.getSymbolTable().setCurrentNode(child);
            child.parseContents(parser);
            // if variable or parameter, add it to scope
            final QName varOrParamName = updateScope(parser, child);
            if (varOrParamName != null) {
                if (locals == null) {
                    locals = new ArrayList<>(2);
                }
                locals.add(varOrParamName);
            }
        }

        parser.getSymbolTable().setCurrentNode(this);

        // after the last element, remove any locals from scope
        if (locals != null) {
            for (QName varOrParamName : locals) {
                parser.removeVariable(varOrParamName);
            }
        }
    }

    /**
     * Add a node to the current scope and return name of a variable or
     * parameter if the node represents a variable or a parameter.
     */
    protected QName updateScope(Parser parser, SyntaxTreeNode node) {
        if (node instanceof Variable) {
            final Variable var = (Variable)node;
            parser.addVariable(var);
            return var.getName();
        }
        else if (node instanceof Param) {
            final Param param = (Param)node;
            parser.addParameter(param);
            return param.getName();
        }
        else {
            return null;
        }
    }

    /**
     * Type check the children of this node. The type check phase may add
     * coercions (CastExpr) to the AST.
     * @param stable The compiler/parser's symbol table
     */
    public abstract Type typeCheck(SymbolTable stable) throws TypeCheckError;

    /**
     * Call typeCheck() on all child syntax tree nodes.
     * @param stable The compiler/parser's symbol table
     */
    protected Type typeCheckContents(SymbolTable stable) throws TypeCheckError {
        for (SyntaxTreeNode item : _contents) {
            item.typeCheck(stable);
        }
        return Type.Void;
    }

    /**
     * Translate this abstract syntax tree node into JVM bytecodes.
     * @param classGen BCEL Java class generator
     * @param methodGen BCEL Java method generator
     */
    public abstract void translate(ClassGenerator classGen,
                                   MethodGenerator methodGen);

    /**
     * Call translate() on all child syntax tree nodes.
     * @param classGen BCEL Java class generator
     * @param methodGen BCEL Java method generator
     */
    protected void translateContents(ClassGenerator classGen,
                                     MethodGenerator methodGen) {
        // Call translate() on all child nodes
        final int n = elementCount();

        for (SyntaxTreeNode item : _contents) {
            methodGen.markChunkStart();
            item.translate(classGen, methodGen);
            methodGen.markChunkEnd();
        }

        // After translation, unmap any registers for any variables/parameters
        // that were declared in this scope. Performing this unmapping in the
        // same AST scope as the declaration deals with the problems of
        // references falling out-of-scope inside the for-each element.
        // (the cause of which being 'lazy' register allocation for references)
        for (int i = 0; i < n; i++) {
            if ( _contents.get(i) instanceof VariableBase) {
                final VariableBase var = (VariableBase)_contents.get(i);
                var.unmapRegister(classGen, methodGen);
            }
        }
    }

    /**
     * Return true if the node represents a simple RTF.
     *
     * A node is a simple RTF if all children only produce Text value.
     *
     * @param node A node
     * @return true if the node content can be considered as a simple RTF.
     */
    private boolean isSimpleRTF(SyntaxTreeNode node) {

        List<SyntaxTreeNode> contents = node.getContents();
        for (SyntaxTreeNode item : contents) {
            if (!isTextElement(item, false))
                return false;
        }

        return true;
    }

     /**
     * Return true if the node represents an adaptive RTF.
     *
     * A node is an adaptive RTF if each children is a Text element
     * or it is <xsl:call-template> or <xsl:apply-templates>.
     *
     * @param node A node
     * @return true if the node content can be considered as an adaptive RTF.
     */
    private boolean isAdaptiveRTF(SyntaxTreeNode node) {

        List<SyntaxTreeNode> contents = node.getContents();
        for (SyntaxTreeNode item : contents) {
            if (!isTextElement(item, true))
                return false;
        }

        return true;
    }

    /**
     * Return true if the node only produces Text content.
     *
     * A node is a Text element if it is Text, xsl:value-of, xsl:number,
     * or a combination of these nested in a control instruction (xsl:if or
     * xsl:choose).
     *
     * If the doExtendedCheck flag is true, xsl:call-template and xsl:apply-templates
     * are also considered as Text elements.
     *
     * @param node A node
     * @param doExtendedCheck If this flag is true, <xsl:call-template> and
     * <xsl:apply-templates> are also considered as Text elements.
     *
     * @return true if the node of Text type
     */
    private boolean isTextElement(SyntaxTreeNode node, boolean doExtendedCheck) {
        if (node instanceof ValueOf || node instanceof Number
            || node instanceof Text)
        {
            return true;
        }
        else if (node instanceof If) {
            return doExtendedCheck ? isAdaptiveRTF(node) : isSimpleRTF(node);
        }
        else if (node instanceof Choose) {
            List<SyntaxTreeNode> contents = node.getContents();
            for (SyntaxTreeNode item : contents) {
                if (item instanceof Text ||
                     ((item instanceof When || item instanceof Otherwise)
                     && ((doExtendedCheck && isAdaptiveRTF(item))
                         || (!doExtendedCheck && isSimpleRTF(item)))))
                    continue;
                else
                    return false;
            }
            return true;
        }
        else if (doExtendedCheck &&
                  (node instanceof CallTemplate
                   || node instanceof ApplyTemplates))
            return true;
        else
            return false;
    }

    /**
     * Utility method used by parameters and variables to store result trees
     * @param classGen BCEL Java class generator
     * @param methodGen BCEL Java method generator
     */
    protected void compileResultTree(ClassGenerator classGen,
                                     MethodGenerator methodGen)
    {
        final ConstantPoolGen cpg = classGen.getConstantPool();
        final InstructionList il = methodGen.getInstructionList();
        final Stylesheet stylesheet = classGen.getStylesheet();

        boolean isSimple = isSimpleRTF(this);
        boolean isAdaptive = false;
        if (!isSimple) {
            isAdaptive = isAdaptiveRTF(this);
        }

        int rtfType = isSimple ? DOM.SIMPLE_RTF
                               : (isAdaptive ? DOM.ADAPTIVE_RTF : DOM.TREE_RTF);

        // Save the current handler base on the stack
        il.append(methodGen.loadHandler());

        final String DOM_CLASS = classGen.getDOMClass();

        // Create new instance of DOM class (with RTF_INITIAL_SIZE nodes)
        //int index = cpg.addMethodref(DOM_IMPL, "<init>", "(I)V");
        //il.append(new NEW(cpg.addClass(DOM_IMPL)));

        il.append(methodGen.loadDOM());
        int index = cpg.addInterfaceMethodref(DOM_INTF,
                                 "getResultTreeFrag",
                                 "(IIZ)" + DOM_INTF_SIG);
        il.append(new PUSH(cpg, RTF_INITIAL_SIZE));
        il.append(new PUSH(cpg, rtfType));
        il.append(new PUSH(cpg, stylesheet.callsNodeset()));
        il.append(new INVOKEINTERFACE(index,4));

        il.append(DUP);

        // Overwrite old handler with DOM handler
        index = cpg.addInterfaceMethodref(DOM_INTF,
                                 "getOutputDomBuilder",
                                 "()" + TRANSLET_OUTPUT_SIG);

        il.append(new INVOKEINTERFACE(index,1));
        il.append(DUP);
        il.append(methodGen.storeHandler());

        // Call startDocument on the new handler
        il.append(methodGen.startDocument());

        // Instantiate result tree fragment
        translateContents(classGen, methodGen);

        // Call endDocument on the new handler
        il.append(methodGen.loadHandler());
        il.append(methodGen.endDocument());

        // Check if we need to wrap the DOMImpl object in a DOMAdapter object.
        // DOMAdapter is not needed if the RTF is a simple RTF and the nodeset()
        // function is not used.
        if (stylesheet.callsNodeset()
            && !DOM_CLASS.equals(DOM_IMPL_CLASS)) {
            // new com.sun.org.apache.xalan.internal.xsltc.dom.DOMAdapter(DOMImpl,String[]);
            index = cpg.addMethodref(DOM_ADAPTER_CLASS,
                                     "<init>",
                                     "("+DOM_INTF_SIG+
                                     "["+STRING_SIG+
                                     "["+STRING_SIG+
                                     "[I"+
                                     "["+STRING_SIG+")V");
            il.append(new NEW(cpg.addClass(DOM_ADAPTER_CLASS)));
            il.append(new DUP_X1());
            il.append(SWAP);

            /*
             * Give the DOM adapter an empty type mapping if the nodeset
             * extension function is never called.
             */
            if (!stylesheet.callsNodeset()) {
                il.append(new ICONST(0));
                il.append(new ANEWARRAY(cpg.addClass(STRING)));
                il.append(DUP);
                il.append(DUP);
                il.append(new ICONST(0));
                il.append(new NEWARRAY(BasicType.INT));
                il.append(SWAP);
                il.append(new INVOKESPECIAL(index));
            }
            else {
                // Push name arrays on the stack
                il.append(ALOAD_0);
                il.append(new GETFIELD(cpg.addFieldref(TRANSLET_CLASS,
                                           NAMES_INDEX,
                                           NAMES_INDEX_SIG)));
                il.append(ALOAD_0);
                il.append(new GETFIELD(cpg.addFieldref(TRANSLET_CLASS,
                                           URIS_INDEX,
                                           URIS_INDEX_SIG)));
                il.append(ALOAD_0);
                il.append(new GETFIELD(cpg.addFieldref(TRANSLET_CLASS,
                                           TYPES_INDEX,
                                           TYPES_INDEX_SIG)));
                il.append(ALOAD_0);
                il.append(new GETFIELD(cpg.addFieldref(TRANSLET_CLASS,
                                           NAMESPACE_INDEX,
                                           NAMESPACE_INDEX_SIG)));

                // Initialized DOM adapter
                il.append(new INVOKESPECIAL(index));

                // Add DOM adapter to MultiDOM class by calling addDOMAdapter()
                il.append(DUP);
                il.append(methodGen.loadDOM());
                il.append(new CHECKCAST(cpg.addClass(classGen.getDOMClass())));
                il.append(SWAP);
                index = cpg.addMethodref(MULTI_DOM_CLASS,
                                         "addDOMAdapter",
                                         "(" + DOM_ADAPTER_SIG + ")I");
                il.append(new INVOKEVIRTUAL(index));
                il.append(POP);         // ignore mask returned by addDOMAdapter
            }
        }

        // Restore old handler base from stack
        il.append(SWAP);
        il.append(methodGen.storeHandler());
    }

    /**
     * Returns true if this expression/instruction depends on the context. By
     * default, every expression/instruction depends on the context unless it
     * overrides this method. Currently used to determine if result trees are
     * compiled using procedures or little DOMs (result tree fragments).
     * @return 'true' if this node depends on the context.
     */
    protected boolean contextDependent() {
        return true;
    }

    /**
     * Return true if any of the expressions/instructions in the contents of
     * this node is context dependent.
     * @return 'true' if the contents of this node is context dependent.
     */
    protected boolean dependentContents() {
        for (SyntaxTreeNode item : _contents) {
            if (item.contextDependent()) {
                return true;
            }
        }
        return false;
    }

    /**
     * Adds a child node to this syntax tree node.
     * @param element is the new child node.
     */
    protected final void addElement(SyntaxTreeNode element) {
        _contents.add(element);
        element.setParent(this);
    }

    /**
     * Inserts the first child node of this syntax tree node. The existing
     * children are shifted back one position.
     * @param element is the new child node.
     */
    protected final void setFirstElement(SyntaxTreeNode element) {
        _contents.add(0, element);
        element.setParent(this);
    }

    /**
     * Removed a child node of this syntax tree node.
     * @param element is the child node to remove.
     */
    protected final void removeElement(SyntaxTreeNode element) {
        _contents.remove(element);
        element.setParent(null);
    }

    /**
     * Returns a List containing all the child nodes of this node.
     * @return A List containing all the child nodes of this node.
     */
    protected final List<SyntaxTreeNode> getContents() {
        return _contents;
    }

    /**
     * Tells you if this node has any child nodes.
     * @return 'true' if this node has any children.
     */
    protected final boolean hasContents() {
        return elementCount() > 0;
    }

    /**
     * Returns the number of children this node has.
     * @return Number of child nodes.
     */
    protected final int elementCount() {
        return _contents.size();
    }

    /**
     * Returns an Iterator of all child nodes of this node.
     * @return An Iterator of all child nodes of this node.
     */
    protected final Iterator<SyntaxTreeNode> elements() {
        return _contents.iterator();
    }

    /**
     * Returns a child node at a given position.
     * @param pos The child node's position.
     * @return The child node.
     */
    protected final SyntaxTreeNode elementAt(int pos) {
        return _contents.get(pos);
    }

    /**
     * Returns this element's last child
     * @return The child node.
     */
    protected final SyntaxTreeNode lastChild() {
        if (_contents.isEmpty()) return null;
        return _contents.get(_contents.size() - 1);
    }

    /**
     * Displays the contents of this syntax tree node (to stdout).
     * This method is intended for debugging _only_, and should be overridden
     * by all syntax tree node implementations.
     * @param indent Indentation level for syntax tree levels.
     */
    public void display(int indent) {
        displayContents(indent);
    }

    /**
     * Displays the contents of this syntax tree node (to stdout).
     * This method is intended for debugging _only_ !!!
     * @param indent Indentation level for syntax tree levels.
     */
    protected void displayContents(int indent) {
        for (SyntaxTreeNode item : _contents) {
            item.display(indent);
        }
    }

    /**
     * Set the indentation level for debug output.
     * @param indent Indentation level for syntax tree levels.
     */
    protected final void indent(int indent) {
        System.out.print(new String(_spaces, 0, indent));
    }

    /**
     * Report an error to the parser.
     * @param element The element in which the error occured (normally 'this'
     * but it could also be an expression/pattern/etc.)
     * @param parser The XSLT parser to report the error to.
     * @param error The error code (from util/ErrorMsg).
     * @param message Any additional error message.
     */
    protected void reportError(SyntaxTreeNode element, Parser parser,
                               String errorCode, String message) {
        final ErrorMsg error = new ErrorMsg(errorCode, message, element);
        parser.reportError(Constants.ERROR, error);
    }

    /**
     * Report a recoverable error to the parser.
     * @param element The element in which the error occured (normally 'this'
     * but it could also be an expression/pattern/etc.)
     * @param parser The XSLT parser to report the error to.
     * @param error The error code (from util/ErrorMsg).
     * @param message Any additional error message.
     */
    protected  void reportWarning(SyntaxTreeNode element, Parser parser,
                                  String errorCode, String message) {
        final ErrorMsg error = new ErrorMsg(errorCode, message, element);
        parser.reportError(Constants.WARNING, error);
    }

}

public interface Constants extends InstructionConstants {

    // Error categories used to report errors to Parser.reportError()

    // Unexpected internal errors, such as null-ptr exceptions, etc.
    // Immediately terminates compilation, no translet produced
    public final int INTERNAL        = 0;
    // XSLT elements that are not implemented and unsupported ext.
    // Immediately terminates compilation, no translet produced
    public final int UNSUPPORTED     = 1;
    // Fatal error in the stylesheet input (parsing or content)
    // Immediately terminates compilation, no translet produced
    public final int FATAL           = 2;
    // Other error in the stylesheet input (parsing or content)
    // Does not terminate compilation, no translet produced
    public final int ERROR           = 3;
    // Other error in the stylesheet input (content errors only)
    // Does not terminate compilation, a translet is produced
    public final int WARNING         = 4;

    public static final String EMPTYSTRING = "";

    public static final String NAMESPACE_FEATURE =
        "http://xml.org/sax/features/namespaces";

    public static final String TRANSLET_INTF
        = "com.sun.org.apache.xalan.internal.xsltc.Translet";
    public static final String TRANSLET_INTF_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/Translet;";

    public static final String ATTRIBUTES_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/runtime/Attributes;";
    public static final String NODE_ITERATOR_SIG
        = "Lcom/sun/org/apache/xml/internal/dtm/DTMAxisIterator;";
    public static final String DOM_INTF_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/DOM;";
    public static final String DOM_IMPL_CLASS
        = "com/sun/org/apache/xalan/internal/xsltc/DOM"; // xml/dtm/ref/DTMDefaultBaseIterators"; //xalan/xsltc/dom/DOMImpl";
        public static final String SAX_IMPL_CLASS
        = "com/sun/org/apache/xalan/internal/xsltc/DOM/SAXImpl";
    public static final String DOM_IMPL_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/SAXImpl;"; //xml/dtm/ref/DTMDefaultBaseIterators"; //xalan/xsltc/dom/DOMImpl;";
        public static final String SAX_IMPL_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/SAXImpl;";
    public static final String DOM_ADAPTER_CLASS
        = "com/sun/org/apache/xalan/internal/xsltc/dom/DOMAdapter";
    public static final String DOM_ADAPTER_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/DOMAdapter;";
    public static final String MULTI_DOM_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.MultiDOM";
    public static final String MULTI_DOM_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/MultiDOM;";

    public static final String STRING
        = "java.lang.String";

    public static final int ACC_PUBLIC
        = com.sun.org.apache.bcel.internal.Constants.ACC_PUBLIC;
    public static final int ACC_SUPER
        = com.sun.org.apache.bcel.internal.Constants.ACC_SUPER;
    public static final int ACC_FINAL
        = com.sun.org.apache.bcel.internal.Constants.ACC_FINAL;
    public static final int ACC_PRIVATE
        = com.sun.org.apache.bcel.internal.Constants.ACC_PRIVATE;
    public static final int ACC_PROTECTED
        = com.sun.org.apache.bcel.internal.Constants.ACC_PROTECTED;
    public static final int ACC_STATIC
        = com.sun.org.apache.bcel.internal.Constants.ACC_STATIC;

    public static final String STRING_SIG
        = "Ljava/lang/String;";
    public static final String STRING_BUFFER_SIG
        = "Ljava/lang/StringBuffer;";
    public static final String OBJECT_SIG
        = "Ljava/lang/Object;";
    public static final String DOUBLE_SIG
        = "Ljava/lang/Double;";
    public static final String INTEGER_SIG
        = "Ljava/lang/Integer;";
    public static final String COLLATOR_CLASS
        = "java/text/Collator";
    public static final String COLLATOR_SIG
        = "Ljava/text/Collator;";

    public static final String NODE
        = "int";
    public static final String NODE_ITERATOR
        = "com.sun.org.apache.xml.internal.dtm.DTMAxisIterator";
    public static final String NODE_ITERATOR_BASE
        = "com.sun.org.apache.xml.internal.dtm.ref.DTMAxisIteratorBase";
    public static final String SORT_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.SortingIterator";
    public static final String SORT_ITERATOR_SIG
        = "Lcom.sun.org.apache.xalan.internal.xsltc.dom.SortingIterator;";
    public static final String NODE_SORT_RECORD
        = "com.sun.org.apache.xalan.internal.xsltc.dom.NodeSortRecord";
    public static final String NODE_SORT_FACTORY
        = "com/sun/org/apache/xalan/internal/xsltc/dom/NodeSortRecordFactory";
    public static final String NODE_SORT_RECORD_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/NodeSortRecord;";
    public static final String NODE_SORT_FACTORY_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/NodeSortRecordFactory;";
    public static final String LOCALE_CLASS
        = "java.util.Locale";
    public static final String LOCALE_SIG
        = "Ljava/util/Locale;";
    public static final String STRING_VALUE_HANDLER
        = "com.sun.org.apache.xalan.internal.xsltc.runtime.StringValueHandler";
    public static final String STRING_VALUE_HANDLER_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/runtime/StringValueHandler;";
    public static final String OUTPUT_HANDLER
        = "com/sun/org/apache/xml/internal/serializer/SerializationHandler";
    public static final String OUTPUT_HANDLER_SIG
        = "Lcom/sun/org/apache/xml/internal/serializer/SerializationHandler;";
    public static final String FILTER_INTERFACE
        = "com.sun.org.apache.xalan.internal.xsltc.dom.Filter";
    public static final String FILTER_INTERFACE_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/Filter;";
    public static final String UNION_ITERATOR_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.UnionIterator";
    public static final String STEP_ITERATOR_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.StepIterator";
    public static final String CACHED_NODE_LIST_ITERATOR_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.CachedNodeListIterator";
    public static final String NTH_ITERATOR_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.NthIterator";
    public static final String ABSOLUTE_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.AbsoluteIterator";
    public static final String DUP_FILTERED_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.DupFilterIterator";
    public static final String CURRENT_NODE_LIST_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.CurrentNodeListIterator";
    public static final String CURRENT_NODE_LIST_FILTER
        = "com.sun.org.apache.xalan.internal.xsltc.dom.CurrentNodeListFilter";
    public static final String CURRENT_NODE_LIST_ITERATOR_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/CurrentNodeListIterator;";
    public static final String CURRENT_NODE_LIST_FILTER_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/CurrentNodeListFilter;";
    public static final String FILTER_STEP_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.FilteredStepIterator";
    public static final String FILTER_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.FilterIterator";
    public static final String SINGLETON_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.SingletonIterator";
    public static final String MATCHING_ITERATOR
        = "com.sun.org.apache.xalan.internal.xsltc.dom.MatchingIterator";
    public static final String NODE_SIG
        = "I";
    public static final String GET_PARENT
        = "getParent";
    public static final String GET_PARENT_SIG
        = "(" + NODE_SIG + ")" + NODE_SIG;
    public static final String NEXT_SIG
        = "()" + NODE_SIG;
    public static final String NEXT
        = "next";
        public static final String NEXTID
        = "nextNodeID";
    public static final String MAKE_NODE
        = "makeNode";
    public static final String MAKE_NODE_LIST
        = "makeNodeList";
    public static final String GET_UNPARSED_ENTITY_URI
        = "getUnparsedEntityURI";
    public static final String STRING_TO_REAL
        = "stringToReal";
    public static final String STRING_TO_REAL_SIG
        = "(" + STRING_SIG + ")D";
    public static final String STRING_TO_INT
        = "stringToInt";
    public static final String STRING_TO_INT_SIG
        = "(" + STRING_SIG + ")I";

    public static final String XSLT_PACKAGE
        = "com.sun.org.apache.xalan.internal.xsltc";
    public static final String COMPILER_PACKAGE
        = XSLT_PACKAGE + ".compiler";
    public static final String RUNTIME_PACKAGE
        = XSLT_PACKAGE + ".runtime";
    public static final String TRANSLET_CLASS
        = RUNTIME_PACKAGE + ".AbstractTranslet";

    public static final String TRANSLET_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/runtime/AbstractTranslet;";
    public static final String UNION_ITERATOR_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/UnionIterator;";
    public static final String TRANSLET_OUTPUT_SIG
        = "Lcom/sun/org/apache/xml/internal/serializer/SerializationHandler;";
    public static final String MAKE_NODE_SIG
        = "(I)Lorg/w3c/dom/Node;";
    public static final String MAKE_NODE_SIG2
        = "(" + NODE_ITERATOR_SIG + ")Lorg/w3c/dom/Node;";
    public static final String MAKE_NODE_LIST_SIG
        = "(I)Lorg/w3c/dom/NodeList;";
    public static final String MAKE_NODE_LIST_SIG2
        = "(" + NODE_ITERATOR_SIG + ")Lorg/w3c/dom/NodeList;";

    public static final String STREAM_XML_OUTPUT
    = "com.sun.org.apache.xml.internal.serializer.ToXMLStream";

    public static final String OUTPUT_BASE
    = "com.sun.org.apache.xml.internal.serializer.SerializerBase";

    public static final String LOAD_DOCUMENT_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.dom.LoadDocument";

    public static final String KEY_INDEX_CLASS
        = "com/sun/org/apache/xalan/internal/xsltc/dom/KeyIndex";
    public static final String KEY_INDEX_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/KeyIndex;";

    public static final String KEY_INDEX_ITERATOR_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/KeyIndex$KeyIndexIterator;";
    public static final String DOM_INTF
        = "com.sun.org.apache.xalan.internal.xsltc.DOM";
    public static final String DOM_IMPL
        = "com.sun.org.apache.xalan.internal.xsltc.dom.SAXImpl";
        public static final String SAX_IMPL
        = "com.sun.org.apache.xalan.internal.xsltc.dom.SAXImpl";
    public static final String STRING_CLASS
        = "java.lang.String";
    public static final String OBJECT_CLASS
        = "java.lang.Object";
    public static final String BOOLEAN_CLASS
        = "java.lang.Boolean";
    public static final String STRING_BUFFER_CLASS
        = "java.lang.StringBuffer";
    public static final String STRING_WRITER
        = "java.io.StringWriter";
    public static final String WRITER_SIG
        = "Ljava/io/Writer;";

    public static final String TRANSLET_OUTPUT_BASE
        = "com.sun.org.apache.xalan.internal.xsltc.TransletOutputBase";
    // output interface
    public static final String TRANSLET_OUTPUT_INTERFACE
        = "com.sun.org.apache.xml.internal.serializer.SerializationHandler";
    public static final String BASIS_LIBRARY_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.runtime.BasisLibrary";
    public static final String ATTRIBUTE_LIST_IMPL_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.runtime.AttributeListImpl";
    public static final String DOUBLE_CLASS
        = "java.lang.Double";
    public static final String INTEGER_CLASS
        = "java.lang.Integer";
    public static final String RUNTIME_NODE_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.runtime.Node";
    public static final String MATH_CLASS
        = "java.lang.Math";

    public static final String BOOLEAN_VALUE
        = "booleanValue";
    public static final String BOOLEAN_VALUE_SIG
        = "()Z";
    public static final String INT_VALUE
        = "intValue";
    public static final String INT_VALUE_SIG
        = "()I";
    public static final String DOUBLE_VALUE
        = "doubleValue";
    public static final String DOUBLE_VALUE_SIG
        = "()D";

    public static final String DOM_PNAME
  = "dom";
    public static final String NODE_PNAME
        = "node";
    public static final String TRANSLET_OUTPUT_PNAME
        = "handler";
    public static final String ITERATOR_PNAME
        = "iterator";
    public static final String DOCUMENT_PNAME
        = "document";
    public static final String TRANSLET_PNAME
        = "translet";

    public static final String INVOKE_METHOD
        = "invokeMethod";
    public static final String GET_NODE_NAME
        = "getNodeNameX";
    public static final String CHARACTERSW
        = "characters";
    public static final String GET_CHILDREN
        = "getChildren";
    public static final String GET_TYPED_CHILDREN
        = "getTypedChildren";
    public static final String CHARACTERS
        = "characters";
    public static final String APPLY_TEMPLATES
        = "applyTemplates";
    public static final String GET_NODE_TYPE
        = "getNodeType";
    public static final String GET_NODE_VALUE
        = "getStringValueX";
    public static final String GET_ELEMENT_VALUE
        = "getElementValue";
    public static final String GET_ATTRIBUTE_VALUE
        = "getAttributeValue";
    public static final String HAS_ATTRIBUTE
        = "hasAttribute";
    public static final String ADD_ITERATOR
        = "addIterator";
    public static final String SET_START_NODE
        = "setStartNode";
    public static final String RESET
        = "reset";

    public static final String ATTR_SET_SIG
        = "(" + DOM_INTF_SIG  + NODE_ITERATOR_SIG + TRANSLET_OUTPUT_SIG + "I)V";

    public static final String GET_NODE_NAME_SIG
        = "(" + NODE_SIG + ")" + STRING_SIG;
    public static final String CHARACTERSW_SIG
        = "("  + STRING_SIG + TRANSLET_OUTPUT_SIG + ")V";
    public static final String CHARACTERS_SIG
        = "(" + NODE_SIG + TRANSLET_OUTPUT_SIG + ")V";
    public static final String GET_CHILDREN_SIG
        = "(" + NODE_SIG +")" + NODE_ITERATOR_SIG;
    public static final String GET_TYPED_CHILDREN_SIG
        = "(I)" + NODE_ITERATOR_SIG;
    public static final String GET_NODE_TYPE_SIG
        = "()S";
    public static final String GET_NODE_VALUE_SIG
        = "(I)" + STRING_SIG;
    public static final String GET_ELEMENT_VALUE_SIG
        = "(I)" + STRING_SIG;
    public static final String GET_ATTRIBUTE_VALUE_SIG
        = "(II)" + STRING_SIG;
    public static final String HAS_ATTRIBUTE_SIG
        = "(II)Z";
    public static final String GET_ITERATOR_SIG
        = "()" + NODE_ITERATOR_SIG;

    public static final String NAMES_INDEX
        = "namesArray";
    public static final String NAMES_INDEX_SIG
        = "[" + STRING_SIG;
    public static final String URIS_INDEX
       = "urisArray";
    public static final String URIS_INDEX_SIG
       = "[" + STRING_SIG;
    public static final String TYPES_INDEX
       = "typesArray";
    public static final String TYPES_INDEX_SIG
       = "[I";
    public static final String NAMESPACE_INDEX
        = "namespaceArray";
    public static final String NAMESPACE_INDEX_SIG
        = "[" + STRING_SIG;
    public static final String HASIDCALL_INDEX
        = "_hasIdCall";
    public static final String HASIDCALL_INDEX_SIG
        = "Z";
    public static final String TRANSLET_VERSION_INDEX
        = "transletVersion";
    public static final String TRANSLET_VERSION_INDEX_SIG
        = "I";

    public static final String DOM_FIELD
        = "_dom";
    public static final String STATIC_NAMES_ARRAY_FIELD
        = "_sNamesArray";
    public static final String STATIC_URIS_ARRAY_FIELD
        = "_sUrisArray";
    public static final String STATIC_TYPES_ARRAY_FIELD
        = "_sTypesArray";
    public static final String STATIC_NAMESPACE_ARRAY_FIELD
        = "_sNamespaceArray";
    public static final String STATIC_CHAR_DATA_FIELD
        = "_scharData";
    public static final String STATIC_CHAR_DATA_FIELD_SIG
        = "[C";
    public static final String FORMAT_SYMBOLS_FIELD
        = "format_symbols";

    public static final String ITERATOR_FIELD_SIG
        = NODE_ITERATOR_SIG;
    public static final String NODE_FIELD
        = "node";
    public static final String NODE_FIELD_SIG
        = "I";

    public static final String EMPTYATTR_FIELD
        = "EmptyAttributes";
    public static final String ATTRIBUTE_LIST_FIELD
        = "attributeList";
    public static final String CLEAR_ATTRIBUTES
        = "clear";
    public static final String ADD_ATTRIBUTE
        = "addAttribute";
    public static final String ATTRIBUTE_LIST_IMPL_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/runtime/AttributeListImpl;";
    public static final String CLEAR_ATTRIBUTES_SIG
        = "()" + ATTRIBUTE_LIST_IMPL_SIG;
    public static final String ADD_ATTRIBUTE_SIG
        = "(" + STRING_SIG + STRING_SIG + ")" + ATTRIBUTE_LIST_IMPL_SIG;

    public static final String ADD_ITERATOR_SIG
        = "(" + NODE_ITERATOR_SIG +")" + UNION_ITERATOR_SIG;

    public static final String ORDER_ITERATOR
        = "orderNodes";
    public static final String ORDER_ITERATOR_SIG
        = "("+NODE_ITERATOR_SIG+"I)"+NODE_ITERATOR_SIG;

    public static final String SET_START_NODE_SIG
        = "(" + NODE_SIG + ")" + NODE_ITERATOR_SIG;

    public static final String NODE_COUNTER
        = "com.sun.org.apache.xalan.internal.xsltc.dom.NodeCounter";
    public static final String NODE_COUNTER_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/NodeCounter;";
    public static final String DEFAULT_NODE_COUNTER
        = "com.sun.org.apache.xalan.internal.xsltc.dom.DefaultNodeCounter";
    public static final String DEFAULT_NODE_COUNTER_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/dom/DefaultNodeCounter;";
    public static final String TRANSLET_FIELD
        = "translet";
    public static final String TRANSLET_FIELD_SIG
        = TRANSLET_SIG;

    public static final String RESET_SIG
        = "()" + NODE_ITERATOR_SIG;
    public static final String GET_PARAMETER
        = "getParameter";
    public static final String ADD_PARAMETER
        = "addParameter";
    public static final String PUSH_PARAM_FRAME
        = "pushParamFrame";
    public static final String PUSH_PARAM_FRAME_SIG
        = "()V";
    public static final String POP_PARAM_FRAME
        = "popParamFrame";
    public static final String POP_PARAM_FRAME_SIG
        = "()V";
    public static final String GET_PARAMETER_SIG
        = "(" + STRING_SIG + ")" + OBJECT_SIG;
    public static final String ADD_PARAMETER_SIG
        = "(" + STRING_SIG + OBJECT_SIG + "Z)" + OBJECT_SIG;

    public static final String STRIP_SPACE
        = "stripSpace";
    public static final String STRIP_SPACE_INTF
        = "com/sun/org/apache/xalan/internal/xsltc/StripFilter";
    public static final String STRIP_SPACE_SIG
        = "Lcom/sun/org/apache/xalan/internal/xsltc/StripFilter;";
    public static final String STRIP_SPACE_PARAMS
        = "(Lcom/sun/org/apache/xalan/internal/xsltc/DOM;II)Z";

    public static final String GET_NODE_VALUE_ITERATOR
        = "getNodeValueIterator";
    public static final String GET_NODE_VALUE_ITERATOR_SIG
        = "("+NODE_ITERATOR_SIG+"I"+STRING_SIG+"Z)"+NODE_ITERATOR_SIG;

    public static final String GET_UNPARSED_ENTITY_URI_SIG
        = "("+STRING_SIG+")"+STRING_SIG;

    public static final int POSITION_INDEX = 2;
    public static final int LAST_INDEX     = 3;

    public static final String XMLNS_PREFIX = "xmlns";
    public static final String XMLNS_STRING = "xmlns:";
    public static final String XMLNS_URI
        = "http://www.w3.org/2000/xmlns/";
    public static final String XSLT_URI
        = "http://www.w3.org/1999/XSL/Transform";
    public static final String XHTML_URI
        = "http://www.w3.org/1999/xhtml";
    public static final String TRANSLET_URI
        = "http://xml.apache.org/xalan/xsltc";
    public static final String REDIRECT_URI
        = "http://xml.apache.org/xalan/redirect";
    public static final String FALLBACK_CLASS
        = "com.sun.org.apache.xalan.internal.xsltc.compiler.Fallback";

    public static final int RTF_INITIAL_SIZE = 32;
}




public interface InstructionConstants {
  public static final Instruction           NOP          = new NOP();
  public static final Instruction           ACONST_NULL  = new ACONST_NULL();
  public static final Instruction           ICONST_M1    = new ICONST(-1);
  public static final Instruction           ICONST_0     = new ICONST(0);
  public static final Instruction           ICONST_1     = new ICONST(1);
  public static final Instruction           ICONST_2     = new ICONST(2);
  public static final Instruction           ICONST_3     = new ICONST(3);
  public static final Instruction           ICONST_4     = new ICONST(4);
  public static final Instruction           ICONST_5     = new ICONST(5);
  public static final Instruction           LCONST_0     = new LCONST(0);
  public static final Instruction           LCONST_1     = new LCONST(1);
  public static final Instruction           FCONST_0     = new FCONST(0);
  public static final Instruction           FCONST_1     = new FCONST(1);
  public static final Instruction           FCONST_2     = new FCONST(2);
  public static final Instruction           DCONST_0     = new DCONST(0);
  public static final Instruction           DCONST_1     = new DCONST(1);
  public static final ArrayInstruction      IALOAD       = new IALOAD();
  public static final ArrayInstruction      LALOAD       = new LALOAD();
  public static final ArrayInstruction      FALOAD       = new FALOAD();
  public static final ArrayInstruction      DALOAD       = new DALOAD();
  public static final ArrayInstruction      AALOAD       = new AALOAD();
  public static final ArrayInstruction      BALOAD       = new BALOAD();
  public static final ArrayInstruction      CALOAD       = new CALOAD();
  public static final ArrayInstruction      SALOAD       = new SALOAD();
  public static final ArrayInstruction      IASTORE      = new IASTORE();
  public static final ArrayInstruction      LASTORE      = new LASTORE();
  public static final ArrayInstruction      FASTORE      = new FASTORE();
  public static final ArrayInstruction      DASTORE      = new DASTORE();
  public static final ArrayInstruction      AASTORE      = new AASTORE();
  public static final ArrayInstruction      BASTORE      = new BASTORE();
  public static final ArrayInstruction      CASTORE      = new CASTORE();
  public static final ArrayInstruction      SASTORE      = new SASTORE();
  public static final StackInstruction      POP          = new POP();
  public static final StackInstruction      POP2         = new POP2();
  public static final StackInstruction      DUP          = new DUP();
  public static final StackInstruction      DUP_X1       = new DUP_X1();
  public static final StackInstruction      DUP_X2       = new DUP_X2();
  public static final StackInstruction      DUP2         = new DUP2();
  public static final StackInstruction      DUP2_X1      = new DUP2_X1();
  public static final StackInstruction      DUP2_X2      = new DUP2_X2();
  public static final StackInstruction      SWAP         = new SWAP();
  public static final ArithmeticInstruction IADD         = new IADD();
  public static final ArithmeticInstruction LADD         = new LADD();
  public static final ArithmeticInstruction FADD         = new FADD();
  public static final ArithmeticInstruction DADD         = new DADD();
  public static final ArithmeticInstruction ISUB         = new ISUB();
  public static final ArithmeticInstruction LSUB         = new LSUB();
  public static final ArithmeticInstruction FSUB         = new FSUB();
  public static final ArithmeticInstruction DSUB         = new DSUB();
  public static final ArithmeticInstruction IMUL         = new IMUL();
  public static final ArithmeticInstruction LMUL         = new LMUL();
  public static final ArithmeticInstruction FMUL         = new FMUL();
  public static final ArithmeticInstruction DMUL         = new DMUL();
  public static final ArithmeticInstruction IDIV         = new IDIV();
  public static final ArithmeticInstruction LDIV         = new LDIV();
  public static final ArithmeticInstruction FDIV         = new FDIV();
  public static final ArithmeticInstruction DDIV         = new DDIV();
  public static final ArithmeticInstruction IREM         = new IREM();
  public static final ArithmeticInstruction LREM         = new LREM();
  public static final ArithmeticInstruction FREM         = new FREM();
  public static final ArithmeticInstruction DREM         = new DREM();
  public static final ArithmeticInstruction INEG         = new INEG();
  public static final ArithmeticInstruction LNEG         = new LNEG();
  public static final ArithmeticInstruction FNEG         = new FNEG();
  public static final ArithmeticInstruction DNEG         = new DNEG();
  public static final ArithmeticInstruction ISHL         = new ISHL();
  public static final ArithmeticInstruction LSHL         = new LSHL();
  public static final ArithmeticInstruction ISHR         = new ISHR();
  public static final ArithmeticInstruction LSHR         = new LSHR();
  public static final ArithmeticInstruction IUSHR        = new IUSHR();
  public static final ArithmeticInstruction LUSHR        = new LUSHR();
  public static final ArithmeticInstruction IAND         = new IAND();
  public static final ArithmeticInstruction LAND         = new LAND();
  public static final ArithmeticInstruction IOR          = new IOR();
  public static final ArithmeticInstruction LOR          = new LOR();
  public static final ArithmeticInstruction IXOR         = new IXOR();
  public static final ArithmeticInstruction LXOR         = new LXOR();
  public static final ConversionInstruction I2L          = new I2L();
  public static final ConversionInstruction I2F          = new I2F();
  public static final ConversionInstruction I2D          = new I2D();
  public static final ConversionInstruction L2I          = new L2I();
  public static final ConversionInstruction L2F          = new L2F();
  public static final ConversionInstruction L2D          = new L2D();
  public static final ConversionInstruction F2I          = new F2I();
  public static final ConversionInstruction F2L          = new F2L();
  public static final ConversionInstruction F2D          = new F2D();
  public static final ConversionInstruction D2I          = new D2I();
  public static final ConversionInstruction D2L          = new D2L();
  public static final ConversionInstruction D2F          = new D2F();
  public static final ConversionInstruction I2B          = new I2B();
  public static final ConversionInstruction I2C          = new I2C();
  public static final ConversionInstruction I2S          = new I2S();
  public static final Instruction           LCMP         = new LCMP();
  public static final Instruction           FCMPL        = new FCMPL();
  public static final Instruction           FCMPG        = new FCMPG();
  public static final Instruction           DCMPL        = new DCMPL();
  public static final Instruction           DCMPG        = new DCMPG();
  public static final ReturnInstruction     IRETURN      = new IRETURN();
  public static final ReturnInstruction     LRETURN      = new LRETURN();
  public static final ReturnInstruction     FRETURN      = new FRETURN();
  public static final ReturnInstruction     DRETURN      = new DRETURN();
  public static final ReturnInstruction     ARETURN      = new ARETURN();
  public static final ReturnInstruction     RETURN       = new RETURN();
  public static final Instruction           ARRAYLENGTH  = new ARRAYLENGTH();
  public static final Instruction           ATHROW       = new ATHROW();
  public static final Instruction           MONITORENTER = new MONITORENTER();
  public static final Instruction           MONITOREXIT  = new MONITOREXIT();

  /** You can use these constants in multiple places safely, if you can guarantee
   * that you will never alter their internal values, e.g. call setIndex().
   */
  public static final LocalVariableInstruction THIS    = new ALOAD(0);
  public static final LocalVariableInstruction ALOAD_0 = THIS;
  public static final LocalVariableInstruction ALOAD_1 = new ALOAD(1);
  public static final LocalVariableInstruction ALOAD_2 = new ALOAD(2);
  public static final LocalVariableInstruction ILOAD_0 = new ILOAD(0);
  public static final LocalVariableInstruction ILOAD_1 = new ILOAD(1);
  public static final LocalVariableInstruction ILOAD_2 = new ILOAD(2);
  public static final LocalVariableInstruction ASTORE_0 = new ASTORE(0);
  public static final LocalVariableInstruction ASTORE_1 = new ASTORE(1);
  public static final LocalVariableInstruction ASTORE_2 = new ASTORE(2);
  public static final LocalVariableInstruction ISTORE_0 = new ISTORE(0);
  public static final LocalVariableInstruction ISTORE_1 = new ISTORE(1);
  public static final LocalVariableInstruction ISTORE_2 = new ISTORE(2);


  /** Get object via its opcode, for immutable instructions like
   * branch instructions entries are set to null.
   */
  public static final Instruction[] INSTRUCTIONS = new Instruction[256];

  /** Interfaces may have no static initializers, so we simulate this
   * with an inner class.
   */
  static final Clinit bla = new Clinit();

  static class Clinit {
    Clinit() {
      INSTRUCTIONS[Constants.NOP] = NOP;
      INSTRUCTIONS[Constants.ACONST_NULL] = ACONST_NULL;
      INSTRUCTIONS[Constants.ICONST_M1] = ICONST_M1;
      INSTRUCTIONS[Constants.ICONST_0] = ICONST_0;
      INSTRUCTIONS[Constants.ICONST_1] = ICONST_1;
      INSTRUCTIONS[Constants.ICONST_2] = ICONST_2;
      INSTRUCTIONS[Constants.ICONST_3] = ICONST_3;
      INSTRUCTIONS[Constants.ICONST_4] = ICONST_4;
      INSTRUCTIONS[Constants.ICONST_5] = ICONST_5;
      INSTRUCTIONS[Constants.LCONST_0] = LCONST_0;
      INSTRUCTIONS[Constants.LCONST_1] = LCONST_1;
      INSTRUCTIONS[Constants.FCONST_0] = FCONST_0;
      INSTRUCTIONS[Constants.FCONST_1] = FCONST_1;
      INSTRUCTIONS[Constants.FCONST_2] = FCONST_2;
      INSTRUCTIONS[Constants.DCONST_0] = DCONST_0;
      INSTRUCTIONS[Constants.DCONST_1] = DCONST_1;
      INSTRUCTIONS[Constants.IALOAD] = IALOAD;
      INSTRUCTIONS[Constants.LALOAD] = LALOAD;
      INSTRUCTIONS[Constants.FALOAD] = FALOAD;
      INSTRUCTIONS[Constants.DALOAD] = DALOAD;
      INSTRUCTIONS[Constants.AALOAD] = AALOAD;
      INSTRUCTIONS[Constants.BALOAD] = BALOAD;
      INSTRUCTIONS[Constants.CALOAD] = CALOAD;
      INSTRUCTIONS[Constants.SALOAD] = SALOAD;
      INSTRUCTIONS[Constants.IASTORE] = IASTORE;
      INSTRUCTIONS[Constants.LASTORE] = LASTORE;
      INSTRUCTIONS[Constants.FASTORE] = FASTORE;
      INSTRUCTIONS[Constants.DASTORE] = DASTORE;
      INSTRUCTIONS[Constants.AASTORE] = AASTORE;
      INSTRUCTIONS[Constants.BASTORE] = BASTORE;
      INSTRUCTIONS[Constants.CASTORE] = CASTORE;
      INSTRUCTIONS[Constants.SASTORE] = SASTORE;
      INSTRUCTIONS[Constants.POP] = POP;
      INSTRUCTIONS[Constants.POP2] = POP2;
      INSTRUCTIONS[Constants.DUP] = DUP;
      INSTRUCTIONS[Constants.DUP_X1] = DUP_X1;
      INSTRUCTIONS[Constants.DUP_X2] = DUP_X2;
      INSTRUCTIONS[Constants.DUP2] = DUP2;
      INSTRUCTIONS[Constants.DUP2_X1] = DUP2_X1;
      INSTRUCTIONS[Constants.DUP2_X2] = DUP2_X2;
      INSTRUCTIONS[Constants.SWAP] = SWAP;
      INSTRUCTIONS[Constants.IADD] = IADD;
      INSTRUCTIONS[Constants.LADD] = LADD;
      INSTRUCTIONS[Constants.FADD] = FADD;
      INSTRUCTIONS[Constants.DADD] = DADD;
      INSTRUCTIONS[Constants.ISUB] = ISUB;
      INSTRUCTIONS[Constants.LSUB] = LSUB;
      INSTRUCTIONS[Constants.FSUB] = FSUB;
      INSTRUCTIONS[Constants.DSUB] = DSUB;
      INSTRUCTIONS[Constants.IMUL] = IMUL;
      INSTRUCTIONS[Constants.LMUL] = LMUL;
      INSTRUCTIONS[Constants.FMUL] = FMUL;
      INSTRUCTIONS[Constants.DMUL] = DMUL;
      INSTRUCTIONS[Constants.IDIV] = IDIV;
      INSTRUCTIONS[Constants.LDIV] = LDIV;
      INSTRUCTIONS[Constants.FDIV] = FDIV;
      INSTRUCTIONS[Constants.DDIV] = DDIV;
      INSTRUCTIONS[Constants.IREM] = IREM;
      INSTRUCTIONS[Constants.LREM] = LREM;
      INSTRUCTIONS[Constants.FREM] = FREM;
      INSTRUCTIONS[Constants.DREM] = DREM;
      INSTRUCTIONS[Constants.INEG] = INEG;
      INSTRUCTIONS[Constants.LNEG] = LNEG;
      INSTRUCTIONS[Constants.FNEG] = FNEG;
      INSTRUCTIONS[Constants.DNEG] = DNEG;
      INSTRUCTIONS[Constants.ISHL] = ISHL;
      INSTRUCTIONS[Constants.LSHL] = LSHL;
      INSTRUCTIONS[Constants.ISHR] = ISHR;
      INSTRUCTIONS[Constants.LSHR] = LSHR;
      INSTRUCTIONS[Constants.IUSHR] = IUSHR;
      INSTRUCTIONS[Constants.LUSHR] = LUSHR;
      INSTRUCTIONS[Constants.IAND] = IAND;
      INSTRUCTIONS[Constants.LAND] = LAND;
      INSTRUCTIONS[Constants.IOR] = IOR;
      INSTRUCTIONS[Constants.LOR] = LOR;
      INSTRUCTIONS[Constants.IXOR] = IXOR;
      INSTRUCTIONS[Constants.LXOR] = LXOR;
      INSTRUCTIONS[Constants.I2L] = I2L;
      INSTRUCTIONS[Constants.I2F] = I2F;
      INSTRUCTIONS[Constants.I2D] = I2D;
      INSTRUCTIONS[Constants.L2I] = L2I;
      INSTRUCTIONS[Constants.L2F] = L2F;
      INSTRUCTIONS[Constants.L2D] = L2D;
      INSTRUCTIONS[Constants.F2I] = F2I;
      INSTRUCTIONS[Constants.F2L] = F2L;
      INSTRUCTIONS[Constants.F2D] = F2D;
      INSTRUCTIONS[Constants.D2I] = D2I;
      INSTRUCTIONS[Constants.D2L] = D2L;
      INSTRUCTIONS[Constants.D2F] = D2F;
      INSTRUCTIONS[Constants.I2B] = I2B;
      INSTRUCTIONS[Constants.I2C] = I2C;
      INSTRUCTIONS[Constants.I2S] = I2S;
      INSTRUCTIONS[Constants.LCMP] = LCMP;
      INSTRUCTIONS[Constants.FCMPL] = FCMPL;
      INSTRUCTIONS[Constants.FCMPG] = FCMPG;
      INSTRUCTIONS[Constants.DCMPL] = DCMPL;
      INSTRUCTIONS[Constants.DCMPG] = DCMPG;
      INSTRUCTIONS[Constants.IRETURN] = IRETURN;
      INSTRUCTIONS[Constants.LRETURN] = LRETURN;
      INSTRUCTIONS[Constants.FRETURN] = FRETURN;
      INSTRUCTIONS[Constants.DRETURN] = DRETURN;
      INSTRUCTIONS[Constants.ARETURN] = ARETURN;
      INSTRUCTIONS[Constants.RETURN] = RETURN;
      INSTRUCTIONS[Constants.ARRAYLENGTH] = ARRAYLENGTH;
      INSTRUCTIONS[Constants.ATHROW] = ATHROW;
      INSTRUCTIONS[Constants.MONITORENTER] = MONITORENTER;
      INSTRUCTIONS[Constants.MONITOREXIT] = MONITOREXIT;
    }
  }
}


/*
 * reserved comment block
 * DO NOT REMOVE OR ALTER!
 */
package com.sun.org.apache.bcel.internal;

/* ====================================================================
 * The Apache Software License, Version 1.1
 *
 * Copyright (c) 2001 The Apache Software Foundation.  All rights
 * reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 *
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in
 *    the documentation and/or other materials provided with the
 *    distribution.
 *
 * 3. The end-user documentation included with the redistribution,
 *    if any, must include the following acknowledgment:
 *       "This product includes software developed by the
 *        Apache Software Foundation (http://www.apache.org/)."
 *    Alternately, this acknowledgment may appear in the software itself,
 *    if and wherever such third-party acknowledgments normally appear.
 *
 * 4. The names "Apache" and "Apache Software Foundation" and
 *    "Apache BCEL" must not be used to endorse or promote products
 *    derived from this software without prior written permission. For
 *    written permission, please contact apache@apache.org.
 *
 * 5. Products derived from this software may not be called "Apache",
 *    "Apache BCEL", nor may "Apache" appear in their name, without
 *    prior written permission of the Apache Software Foundation.
 *
 * THIS SOFTWARE IS PROVIDED ``AS IS'' AND ANY EXPRESSED OR IMPLIED
 * WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
 * DISCLAIMED.  IN NO EVENT SHALL THE APACHE SOFTWARE FOUNDATION OR
 * ITS CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
 * LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF
 * USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
 * ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
 * OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT
 * OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 * ====================================================================
 *
 * This software consists of voluntary contributions made by many
 * individuals on behalf of the Apache Software Foundation.  For more
 * information on the Apache Software Foundation, please see
 * <http://www.apache.org/>.
 */

/**
 * Constants for the project, mostly defined in the JVM specification.
 *
 * @author  <A HREF="mailto:markus.dahm@berlin.de">M. Dahm</A>
 */
public interface Constants {
  /** Major and minor version of the code.
   */
  public final static short MAJOR_1_1 = 45;
  public final static short MINOR_1_1 = 3;
  public final static short MAJOR_1_2 = 46;
  public final static short MINOR_1_2 = 0;
  public final static short MAJOR_1_3 = 47;
  public final static short MINOR_1_3 = 0;
  public final static short MAJOR     = MAJOR_1_1; // Defaults
  public final static short MINOR     = MINOR_1_1;

  /** Maximum value for an unsigned short.
   */
  public final static int MAX_SHORT = 65535; // 2^16 - 1

  /** Maximum value for an unsigned byte.
   */
  public final static int MAX_BYTE  = 255; // 2^8 - 1

  /** Access flags for classes, fields and methods.
   */
  public final static short ACC_PUBLIC       = 0x0001;
  public final static short ACC_PRIVATE      = 0x0002;
  public final static short ACC_PROTECTED    = 0x0004;
  public final static short ACC_STATIC       = 0x0008;

  public final static short ACC_FINAL        = 0x0010;
  public final static short ACC_SYNCHRONIZED = 0x0020;
  public final static short ACC_VOLATILE     = 0x0040;
  public final static short ACC_TRANSIENT    = 0x0080;

  public final static short ACC_NATIVE       = 0x0100;
  public final static short ACC_INTERFACE    = 0x0200;
  public final static short ACC_ABSTRACT     = 0x0400;
  public final static short ACC_STRICT       = 0x0800;

  // Applies to classes compiled by new compilers only
  public final static short ACC_SUPER        = 0x0020;

  public final static short MAX_ACC_FLAG     = ACC_STRICT;

  public final static String[] ACCESS_NAMES = {
    "public", "private", "protected", "static", "final", "synchronized",
    "volatile", "transient", "native", "interface", "abstract", "strictfp"
  };

  /** Tags in constant pool to denote type of constant.
   */
  public final static byte CONSTANT_Utf8               = 1;
  public final static byte CONSTANT_Integer            = 3;
  public final static byte CONSTANT_Float              = 4;
  public final static byte CONSTANT_Long               = 5;
  public final static byte CONSTANT_Double             = 6;
  public final static byte CONSTANT_Class              = 7;
  public final static byte CONSTANT_Fieldref           = 9;
  public final static byte CONSTANT_String             = 8;
  public final static byte CONSTANT_Methodref          = 10;
  public final static byte CONSTANT_InterfaceMethodref = 11;
  public final static byte CONSTANT_NameAndType        = 12;

  public final static String[] CONSTANT_NAMES = {
    "", "CONSTANT_Utf8", "", "CONSTANT_Integer",
    "CONSTANT_Float", "CONSTANT_Long", "CONSTANT_Double",
    "CONSTANT_Class", "CONSTANT_String", "CONSTANT_Fieldref",
    "CONSTANT_Methodref", "CONSTANT_InterfaceMethodref",
    "CONSTANT_NameAndType" };

  /** The name of the static initializer, also called &quot;class
   *  initialization method&quot; or &quot;interface initialization
   *   method&quot;. This is &quot;&lt;clinit&gt;&quot;.
   */
  public final static String STATIC_INITIALIZER_NAME = "<clinit>";

  /** The name of every constructor method in a class, also called
   * &quot;instance initialization method&quot;. This is &quot;&lt;init&gt;&quot;.
   */
  public final static String CONSTRUCTOR_NAME = "<init>";

  /** The names of the interfaces implemented by arrays */
  public final static String[] INTERFACES_IMPLEMENTED_BY_ARRAYS = {"java.lang.Cloneable", "java.io.Serializable"};

  /**
   * Limitations of the Java Virtual Machine.
   * See The Java Virtual Machine Specification, Second Edition, page 152, chapter 4.10.
   */
  public static final int MAX_CP_ENTRIES     = 65535;
  public static final int MAX_CODE_SIZE      = 65536; //bytes

  /** Java VM opcodes.
   */
  public static final short NOP              = 0;
  public static final short ACONST_NULL      = 1;
  public static final short ICONST_M1        = 2;
  public static final short ICONST_0         = 3;
  public static final short ICONST_1         = 4;
  public static final short ICONST_2         = 5;
  public static final short ICONST_3         = 6;
  public static final short ICONST_4         = 7;
  public static final short ICONST_5         = 8;
  public static final short LCONST_0         = 9;
  public static final short LCONST_1         = 10;
  public static final short FCONST_0         = 11;
  public static final short FCONST_1         = 12;
  public static final short FCONST_2         = 13;
  public static final short DCONST_0         = 14;
  public static final short DCONST_1         = 15;
  public static final short BIPUSH           = 16;
  public static final short SIPUSH           = 17;
  public static final short LDC              = 18;
  public static final short LDC_W            = 19;
  public static final short LDC2_W           = 20;
  public static final short ILOAD            = 21;
  public static final short LLOAD            = 22;
  public static final short FLOAD            = 23;
  public static final short DLOAD            = 24;
  public static final short ALOAD            = 25;
  public static final short ILOAD_0          = 26;
  public static final short ILOAD_1          = 27;
  public static final short ILOAD_2          = 28;
  public static final short ILOAD_3          = 29;
  public static final short LLOAD_0          = 30;
  public static final short LLOAD_1          = 31;
  public static final short LLOAD_2          = 32;
  public static final short LLOAD_3          = 33;
  public static final short FLOAD_0          = 34;
  public static final short FLOAD_1          = 35;
  public static final short FLOAD_2          = 36;
  public static final short FLOAD_3          = 37;
  public static final short DLOAD_0          = 38;
  public static final short DLOAD_1          = 39;
  public static final short DLOAD_2          = 40;
  public static final short DLOAD_3          = 41;
  public static final short ALOAD_0          = 42;
  public static final short ALOAD_1          = 43;
  public static final short ALOAD_2          = 44;
  public static final short ALOAD_3          = 45;
  public static final short IALOAD           = 46;
  public static final short LALOAD           = 47;
  public static final short FALOAD           = 48;
  public static final short DALOAD           = 49;
  public static final short AALOAD           = 50;
  public static final short BALOAD           = 51;
  public static final short CALOAD           = 52;
  public static final short SALOAD           = 53;
  public static final short ISTORE           = 54;
  public static final short LSTORE           = 55;
  public static final short FSTORE           = 56;
  public static final short DSTORE           = 57;
  public static final short ASTORE           = 58;
  public static final short ISTORE_0         = 59;
  public static final short ISTORE_1         = 60;
  public static final short ISTORE_2         = 61;
  public static final short ISTORE_3         = 62;
  public static final short LSTORE_0         = 63;
  public static final short LSTORE_1         = 64;
  public static final short LSTORE_2         = 65;
  public static final short LSTORE_3         = 66;
  public static final short FSTORE_0         = 67;
  public static final short FSTORE_1         = 68;
  public static final short FSTORE_2         = 69;
  public static final short FSTORE_3         = 70;
  public static final short DSTORE_0         = 71;
  public static final short DSTORE_1         = 72;
  public static final short DSTORE_2         = 73;
  public static final short DSTORE_3         = 74;
  public static final short ASTORE_0         = 75;
  public static final short ASTORE_1         = 76;
  public static final short ASTORE_2         = 77;
  public static final short ASTORE_3         = 78;
  public static final short IASTORE          = 79;
  public static final short LASTORE          = 80;
  public static final short FASTORE          = 81;
  public static final short DASTORE          = 82;
  public static final short AASTORE          = 83;
  public static final short BASTORE          = 84;
  public static final short CASTORE          = 85;
  public static final short SASTORE          = 86;
  public static final short POP              = 87;
  public static final short POP2             = 88;
  public static final short DUP              = 89;
  public static final short DUP_X1           = 90;
  public static final short DUP_X2           = 91;
  public static final short DUP2             = 92;
  public static final short DUP2_X1          = 93;
  public static final short DUP2_X2          = 94;
  public static final short SWAP             = 95;
  public static final short IADD             = 96;
  public static final short LADD             = 97;
  public static final short FADD             = 98;
  public static final short DADD             = 99;
  public static final short ISUB             = 100;
  public static final short LSUB             = 101;
  public static final short FSUB             = 102;
  public static final short DSUB             = 103;
  public static final short IMUL             = 104;
  public static final short LMUL             = 105;
  public static final short FMUL             = 106;
  public static final short DMUL             = 107;
  public static final short IDIV             = 108;
  public static final short LDIV             = 109;
  public static final short FDIV             = 110;
  public static final short DDIV             = 111;
  public static final short IREM             = 112;
  public static final short LREM             = 113;
  public static final short FREM             = 114;
  public static final short DREM             = 115;
  public static final short INEG             = 116;
  public static final short LNEG             = 117;
  public static final short FNEG             = 118;
  public static final short DNEG             = 119;
  public static final short ISHL             = 120;
  public static final short LSHL             = 121;
  public static final short ISHR             = 122;
  public static final short LSHR             = 123;
  public static final short IUSHR            = 124;
  public static final short LUSHR            = 125;
  public static final short IAND             = 126;
  public static final short LAND             = 127;
  public static final short IOR              = 128;
  public static final short LOR              = 129;
  public static final short IXOR             = 130;
  public static final short LXOR             = 131;
  public static final short IINC             = 132;
  public static final short I2L              = 133;
  public static final short I2F              = 134;
  public static final short I2D              = 135;
  public static final short L2I              = 136;
  public static final short L2F              = 137;
  public static final short L2D              = 138;
  public static final short F2I              = 139;
  public static final short F2L              = 140;
  public static final short F2D              = 141;
  public static final short D2I              = 142;
  public static final short D2L              = 143;
  public static final short D2F              = 144;
  public static final short I2B              = 145;
  public static final short INT2BYTE         = 145; // Old notion
  public static final short I2C              = 146;
  public static final short INT2CHAR         = 146; // Old notion
  public static final short I2S              = 147;
  public static final short INT2SHORT        = 147; // Old notion
  public static final short LCMP             = 148;
  public static final short FCMPL            = 149;
  public static final short FCMPG            = 150;
  public static final short DCMPL            = 151;
  public static final short DCMPG            = 152;
  public static final short IFEQ             = 153;
  public static final short IFNE             = 154;
  public static final short IFLT             = 155;
  public static final short IFGE             = 156;
  public static final short IFGT             = 157;
  public static final short IFLE             = 158;
  public static final short IF_ICMPEQ        = 159;
  public static final short IF_ICMPNE        = 160;
  public static final short IF_ICMPLT        = 161;
  public static final short IF_ICMPGE        = 162;
  public static final short IF_ICMPGT        = 163;
  public static final short IF_ICMPLE        = 164;
  public static final short IF_ACMPEQ        = 165;
  public static final short IF_ACMPNE        = 166;
  public static final short GOTO             = 167;
  public static final short JSR              = 168;
  public static final short RET              = 169;
  public static final short TABLESWITCH      = 170;
  public static final short LOOKUPSWITCH     = 171;
  public static final short IRETURN          = 172;
  public static final short LRETURN          = 173;
  public static final short FRETURN          = 174;
  public static final short DRETURN          = 175;
  public static final short ARETURN          = 176;
  public static final short RETURN           = 177;
  public static final short GETSTATIC        = 178;
  public static final short PUTSTATIC        = 179;
  public static final short GETFIELD         = 180;
  public static final short PUTFIELD         = 181;
  public static final short INVOKEVIRTUAL    = 182;
  public static final short INVOKESPECIAL    = 183;
  public static final short INVOKENONVIRTUAL = 183; // Old name in JDK 1.0
  public static final short INVOKESTATIC     = 184;
  public static final short INVOKEINTERFACE  = 185;
  public static final short NEW              = 187;
  public static final short NEWARRAY         = 188;
  public static final short ANEWARRAY        = 189;
  public static final short ARRAYLENGTH      = 190;
  public static final short ATHROW           = 191;
  public static final short CHECKCAST        = 192;
  public static final short INSTANCEOF       = 193;
  public static final short MONITORENTER     = 194;
  public static final short MONITOREXIT      = 195;
  public static final short WIDE             = 196;
  public static final short MULTIANEWARRAY   = 197;
  public static final short IFNULL           = 198;
  public static final short IFNONNULL        = 199;
  public static final short GOTO_W           = 200;
  public static final short JSR_W            = 201;

  /**
   * Non-legal opcodes, may be used by JVM internally.
   */
  public static final short BREAKPOINT                = 202;
  public static final short LDC_QUICK                 = 203;
  public static final short LDC_W_QUICK               = 204;
  public static final short LDC2_W_QUICK              = 205;
  public static final short GETFIELD_QUICK            = 206;
  public static final short PUTFIELD_QUICK            = 207;
  public static final short GETFIELD2_QUICK           = 208;
  public static final short PUTFIELD2_QUICK           = 209;
  public static final short GETSTATIC_QUICK           = 210;
  public static final short PUTSTATIC_QUICK           = 211;
  public static final short GETSTATIC2_QUICK          = 212;
  public static final short PUTSTATIC2_QUICK          = 213;
  public static final short INVOKEVIRTUAL_QUICK       = 214;
  public static final short INVOKENONVIRTUAL_QUICK    = 215;
  public static final short INVOKESUPER_QUICK         = 216;
  public static final short INVOKESTATIC_QUICK        = 217;
  public static final short INVOKEINTERFACE_QUICK     = 218;
  public static final short INVOKEVIRTUALOBJECT_QUICK = 219;
  public static final short NEW_QUICK                 = 221;
  public static final short ANEWARRAY_QUICK           = 222;
  public static final short MULTIANEWARRAY_QUICK      = 223;
  public static final short CHECKCAST_QUICK           = 224;
  public static final short INSTANCEOF_QUICK          = 225;
  public static final short INVOKEVIRTUAL_QUICK_W     = 226;
  public static final short GETFIELD_QUICK_W          = 227;
  public static final short PUTFIELD_QUICK_W          = 228;
  public static final short IMPDEP1                   = 254;
  public static final short IMPDEP2                   = 255;

  /**
   * For internal purposes only.
   */
  public static final short PUSH             = 4711;
  public static final short SWITCH           = 4712;

  /**
   * Illegal codes
   */
  public static final short  UNDEFINED      = -1;
  public static final short  UNPREDICTABLE  = -2;
  public static final short  RESERVED       = -3;
  public static final String ILLEGAL_OPCODE = "<illegal opcode>";
  public static final String ILLEGAL_TYPE   = "<illegal type>";

  public static final byte T_BOOLEAN = 4;
  public static final byte T_CHAR    = 5;
  public static final byte T_FLOAT   = 6;
  public static final byte T_DOUBLE  = 7;
  public static final byte T_BYTE    = 8;
  public static final byte T_SHORT   = 9;
  public static final byte T_INT     = 10;
  public static final byte T_LONG    = 11;

  public static final byte T_VOID      = 12; // Non-standard
  public static final byte T_ARRAY     = 13;
  public static final byte T_OBJECT    = 14;
  public static final byte T_REFERENCE = 14; // Deprecated
  public static final byte T_UNKNOWN   = 15;
  public static final byte T_ADDRESS   = 16;

  /** The primitive type names corresponding to the T_XX constants,
   * e.g., TYPE_NAMES[T_INT] = "int"
   */
  public static final String[] TYPE_NAMES = {
    ILLEGAL_TYPE, ILLEGAL_TYPE,  ILLEGAL_TYPE, ILLEGAL_TYPE,
    "boolean", "char", "float", "double", "byte", "short", "int", "long",
    "void", "array", "object", "unknown" // Non-standard
  };

  /** The primitive class names corresponding to the T_XX constants,
   * e.g., CLASS_TYPE_NAMES[T_INT] = "java.lang.Integer"
   */
  public static final String[] CLASS_TYPE_NAMES = {
    ILLEGAL_TYPE, ILLEGAL_TYPE,  ILLEGAL_TYPE, ILLEGAL_TYPE,
    "java.lang.Boolean", "java.lang.Character", "java.lang.Float",
    "java.lang.Double", "java.lang.Byte", "java.lang.Short",
    "java.lang.Integer", "java.lang.Long", "java.lang.Void",
    ILLEGAL_TYPE, ILLEGAL_TYPE,  ILLEGAL_TYPE
  };

  /** The signature characters corresponding to primitive types,
   * e.g., SHORT_TYPE_NAMES[T_INT] = "I"
   */
  public static final String[] SHORT_TYPE_NAMES = {
    ILLEGAL_TYPE, ILLEGAL_TYPE,  ILLEGAL_TYPE, ILLEGAL_TYPE,
    "Z", "C", "F", "D", "B", "S", "I", "J",
    "V", ILLEGAL_TYPE, ILLEGAL_TYPE, ILLEGAL_TYPE
  };

  /**
   * Number of byte code operands, i.e., number of bytes after the tag byte
   * itself.
   */
  public static final short[] NO_OF_OPERANDS = {
    0/*nop*/, 0/*aconst_null*/, 0/*iconst_m1*/, 0/*iconst_0*/,
    0/*iconst_1*/, 0/*iconst_2*/, 0/*iconst_3*/, 0/*iconst_4*/,
    0/*iconst_5*/, 0/*lconst_0*/, 0/*lconst_1*/, 0/*fconst_0*/,
    0/*fconst_1*/, 0/*fconst_2*/, 0/*dconst_0*/, 0/*dconst_1*/,
    1/*bipush*/, 2/*sipush*/, 1/*ldc*/, 2/*ldc_w*/, 2/*ldc2_w*/,
    1/*iload*/, 1/*lload*/, 1/*fload*/, 1/*dload*/, 1/*aload*/,
    0/*iload_0*/, 0/*iload_1*/, 0/*iload_2*/, 0/*iload_3*/,
    0/*lload_0*/, 0/*lload_1*/, 0/*lload_2*/, 0/*lload_3*/,
    0/*fload_0*/, 0/*fload_1*/, 0/*fload_2*/, 0/*fload_3*/,
    0/*dload_0*/, 0/*dload_1*/, 0/*dload_2*/, 0/*dload_3*/,
    0/*aload_0*/, 0/*aload_1*/, 0/*aload_2*/, 0/*aload_3*/,
    0/*iaload*/, 0/*laload*/, 0/*faload*/, 0/*daload*/,
    0/*aaload*/, 0/*baload*/, 0/*caload*/, 0/*saload*/,
    1/*istore*/, 1/*lstore*/, 1/*fstore*/, 1/*dstore*/,
    1/*astore*/, 0/*istore_0*/, 0/*istore_1*/, 0/*istore_2*/,
    0/*istore_3*/, 0/*lstore_0*/, 0/*lstore_1*/, 0/*lstore_2*/,
    0/*lstore_3*/, 0/*fstore_0*/, 0/*fstore_1*/, 0/*fstore_2*/,
    0/*fstore_3*/, 0/*dstore_0*/, 0/*dstore_1*/, 0/*dstore_2*/,
    0/*dstore_3*/, 0/*astore_0*/, 0/*astore_1*/, 0/*astore_2*/,
    0/*astore_3*/, 0/*iastore*/, 0/*lastore*/, 0/*fastore*/,
    0/*dastore*/, 0/*aastore*/, 0/*bastore*/, 0/*castore*/,
    0/*sastore*/, 0/*pop*/, 0/*pop2*/, 0/*dup*/, 0/*dup_x1*/,
    0/*dup_x2*/, 0/*dup2*/, 0/*dup2_x1*/, 0/*dup2_x2*/, 0/*swap*/,
    0/*iadd*/, 0/*ladd*/, 0/*fadd*/, 0/*dadd*/, 0/*isub*/,
    0/*lsub*/, 0/*fsub*/, 0/*dsub*/, 0/*imul*/, 0/*lmul*/,
    0/*fmul*/, 0/*dmul*/, 0/*idiv*/, 0/*ldiv*/, 0/*fdiv*/,
    0/*ddiv*/, 0/*irem*/, 0/*lrem*/, 0/*frem*/, 0/*drem*/,
    0/*ineg*/, 0/*lneg*/, 0/*fneg*/, 0/*dneg*/, 0/*ishl*/,
    0/*lshl*/, 0/*ishr*/, 0/*lshr*/, 0/*iushr*/, 0/*lushr*/,
    0/*iand*/, 0/*land*/, 0/*ior*/, 0/*lor*/, 0/*ixor*/, 0/*lxor*/,
    2/*iinc*/, 0/*i2l*/, 0/*i2f*/, 0/*i2d*/, 0/*l2i*/, 0/*l2f*/,
    0/*l2d*/, 0/*f2i*/, 0/*f2l*/, 0/*f2d*/, 0/*d2i*/, 0/*d2l*/,
    0/*d2f*/, 0/*i2b*/, 0/*i2c*/, 0/*i2s*/, 0/*lcmp*/, 0/*fcmpl*/,
    0/*fcmpg*/, 0/*dcmpl*/, 0/*dcmpg*/, 2/*ifeq*/, 2/*ifne*/,
    2/*iflt*/, 2/*ifge*/, 2/*ifgt*/, 2/*ifle*/, 2/*if_icmpeq*/,
    2/*if_icmpne*/, 2/*if_icmplt*/, 2/*if_icmpge*/, 2/*if_icmpgt*/,
    2/*if_icmple*/, 2/*if_acmpeq*/, 2/*if_acmpne*/, 2/*goto*/,
    2/*jsr*/, 1/*ret*/, UNPREDICTABLE/*tableswitch*/, UNPREDICTABLE/*lookupswitch*/,
    0/*ireturn*/, 0/*lreturn*/, 0/*freturn*/,
    0/*dreturn*/, 0/*areturn*/, 0/*return*/,
    2/*getstatic*/, 2/*putstatic*/, 2/*getfield*/,
    2/*putfield*/, 2/*invokevirtual*/, 2/*invokespecial*/, 2/*invokestatic*/,
    4/*invokeinterface*/, UNDEFINED, 2/*new*/,
    1/*newarray*/, 2/*anewarray*/,
    0/*arraylength*/, 0/*athrow*/, 2/*checkcast*/,
    2/*instanceof*/, 0/*monitorenter*/,
    0/*monitorexit*/, UNPREDICTABLE/*wide*/, 3/*multianewarray*/,
    2/*ifnull*/, 2/*ifnonnull*/, 4/*goto_w*/,
    4/*jsr_w*/, 0/*breakpoint*/, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, RESERVED/*impdep1*/, RESERVED/*impdep2*/
  };

  /**
   * How the byte code operands are to be interpreted.
   */
  public static final short[][] TYPE_OF_OPERANDS = {
    {}/*nop*/, {}/*aconst_null*/, {}/*iconst_m1*/, {}/*iconst_0*/,
    {}/*iconst_1*/, {}/*iconst_2*/, {}/*iconst_3*/, {}/*iconst_4*/,
    {}/*iconst_5*/, {}/*lconst_0*/, {}/*lconst_1*/, {}/*fconst_0*/,
    {}/*fconst_1*/, {}/*fconst_2*/, {}/*dconst_0*/, {}/*dconst_1*/,
    {T_BYTE}/*bipush*/, {T_SHORT}/*sipush*/, {T_BYTE}/*ldc*/,
    {T_SHORT}/*ldc_w*/, {T_SHORT}/*ldc2_w*/,
    {T_BYTE}/*iload*/, {T_BYTE}/*lload*/, {T_BYTE}/*fload*/,
    {T_BYTE}/*dload*/, {T_BYTE}/*aload*/, {}/*iload_0*/,
    {}/*iload_1*/, {}/*iload_2*/, {}/*iload_3*/, {}/*lload_0*/,
    {}/*lload_1*/, {}/*lload_2*/, {}/*lload_3*/, {}/*fload_0*/,
    {}/*fload_1*/, {}/*fload_2*/, {}/*fload_3*/, {}/*dload_0*/,
    {}/*dload_1*/, {}/*dload_2*/, {}/*dload_3*/, {}/*aload_0*/,
    {}/*aload_1*/, {}/*aload_2*/, {}/*aload_3*/, {}/*iaload*/,
    {}/*laload*/, {}/*faload*/, {}/*daload*/, {}/*aaload*/,
    {}/*baload*/, {}/*caload*/, {}/*saload*/, {T_BYTE}/*istore*/,
    {T_BYTE}/*lstore*/, {T_BYTE}/*fstore*/, {T_BYTE}/*dstore*/,
    {T_BYTE}/*astore*/, {}/*istore_0*/, {}/*istore_1*/,
    {}/*istore_2*/, {}/*istore_3*/, {}/*lstore_0*/, {}/*lstore_1*/,
    {}/*lstore_2*/, {}/*lstore_3*/, {}/*fstore_0*/, {}/*fstore_1*/,
    {}/*fstore_2*/, {}/*fstore_3*/, {}/*dstore_0*/, {}/*dstore_1*/,
    {}/*dstore_2*/, {}/*dstore_3*/, {}/*astore_0*/, {}/*astore_1*/,
    {}/*astore_2*/, {}/*astore_3*/, {}/*iastore*/, {}/*lastore*/,
    {}/*fastore*/, {}/*dastore*/, {}/*aastore*/, {}/*bastore*/,
    {}/*castore*/, {}/*sastore*/, {}/*pop*/, {}/*pop2*/, {}/*dup*/,
    {}/*dup_x1*/, {}/*dup_x2*/, {}/*dup2*/, {}/*dup2_x1*/,
    {}/*dup2_x2*/, {}/*swap*/, {}/*iadd*/, {}/*ladd*/, {}/*fadd*/,
    {}/*dadd*/, {}/*isub*/, {}/*lsub*/, {}/*fsub*/, {}/*dsub*/,
    {}/*imul*/, {}/*lmul*/, {}/*fmul*/, {}/*dmul*/, {}/*idiv*/,
    {}/*ldiv*/, {}/*fdiv*/, {}/*ddiv*/, {}/*irem*/, {}/*lrem*/,
    {}/*frem*/, {}/*drem*/, {}/*ineg*/, {}/*lneg*/, {}/*fneg*/,
    {}/*dneg*/, {}/*ishl*/, {}/*lshl*/, {}/*ishr*/, {}/*lshr*/,
    {}/*iushr*/, {}/*lushr*/, {}/*iand*/, {}/*land*/, {}/*ior*/,
    {}/*lor*/, {}/*ixor*/, {}/*lxor*/, {T_BYTE, T_BYTE}/*iinc*/,
    {}/*i2l*/, {}/*i2f*/, {}/*i2d*/, {}/*l2i*/, {}/*l2f*/, {}/*l2d*/,
    {}/*f2i*/, {}/*f2l*/, {}/*f2d*/, {}/*d2i*/, {}/*d2l*/, {}/*d2f*/,
    {}/*i2b*/, {}/*i2c*/,{}/*i2s*/, {}/*lcmp*/, {}/*fcmpl*/,
    {}/*fcmpg*/, {}/*dcmpl*/, {}/*dcmpg*/, {T_SHORT}/*ifeq*/,
    {T_SHORT}/*ifne*/, {T_SHORT}/*iflt*/, {T_SHORT}/*ifge*/,
    {T_SHORT}/*ifgt*/, {T_SHORT}/*ifle*/, {T_SHORT}/*if_icmpeq*/,
    {T_SHORT}/*if_icmpne*/, {T_SHORT}/*if_icmplt*/,
    {T_SHORT}/*if_icmpge*/, {T_SHORT}/*if_icmpgt*/,
    {T_SHORT}/*if_icmple*/, {T_SHORT}/*if_acmpeq*/,
    {T_SHORT}/*if_acmpne*/, {T_SHORT}/*goto*/, {T_SHORT}/*jsr*/,
    {T_BYTE}/*ret*/, {}/*tableswitch*/, {}/*lookupswitch*/,
    {}/*ireturn*/, {}/*lreturn*/, {}/*freturn*/, {}/*dreturn*/,
    {}/*areturn*/, {}/*return*/, {T_SHORT}/*getstatic*/,
    {T_SHORT}/*putstatic*/, {T_SHORT}/*getfield*/,
    {T_SHORT}/*putfield*/, {T_SHORT}/*invokevirtual*/,
    {T_SHORT}/*invokespecial*/, {T_SHORT}/*invokestatic*/,
    {T_SHORT, T_BYTE, T_BYTE}/*invokeinterface*/, {},
    {T_SHORT}/*new*/, {T_BYTE}/*newarray*/,
    {T_SHORT}/*anewarray*/, {}/*arraylength*/, {}/*athrow*/,
    {T_SHORT}/*checkcast*/, {T_SHORT}/*instanceof*/,
    {}/*monitorenter*/, {}/*monitorexit*/, {T_BYTE}/*wide*/,
    {T_SHORT, T_BYTE}/*multianewarray*/, {T_SHORT}/*ifnull*/,
    {T_SHORT}/*ifnonnull*/, {T_INT}/*goto_w*/, {T_INT}/*jsr_w*/,
    {}/*breakpoint*/, {}, {}, {}, {}, {}, {}, {},
    {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {},
    {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {},
    {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {}, {},
    {}/*impdep1*/, {}/*impdep2*/
  };

  /**
   * Names of opcodes.
   */
  public static final String[] OPCODE_NAMES = {
    "nop", "aconst_null", "iconst_m1", "iconst_0", "iconst_1",
    "iconst_2", "iconst_3", "iconst_4", "iconst_5", "lconst_0",
    "lconst_1", "fconst_0", "fconst_1", "fconst_2", "dconst_0",
    "dconst_1", "bipush", "sipush", "ldc", "ldc_w", "ldc2_w", "iload",
    "lload", "fload", "dload", "aload", "iload_0", "iload_1", "iload_2",
    "iload_3", "lload_0", "lload_1", "lload_2", "lload_3", "fload_0",
    "fload_1", "fload_2", "fload_3", "dload_0", "dload_1", "dload_2",
    "dload_3", "aload_0", "aload_1", "aload_2", "aload_3", "iaload",
    "laload", "faload", "daload", "aaload", "baload", "caload", "saload",
    "istore", "lstore", "fstore", "dstore", "astore", "istore_0",
    "istore_1", "istore_2", "istore_3", "lstore_0", "lstore_1",
    "lstore_2", "lstore_3", "fstore_0", "fstore_1", "fstore_2",
    "fstore_3", "dstore_0", "dstore_1", "dstore_2", "dstore_3",
    "astore_0", "astore_1", "astore_2", "astore_3", "iastore", "lastore",
    "fastore", "dastore", "aastore", "bastore", "castore", "sastore",
    "pop", "pop2", "dup", "dup_x1", "dup_x2", "dup2", "dup2_x1",
    "dup2_x2", "swap", "iadd", "ladd", "fadd", "dadd", "isub", "lsub",
    "fsub", "dsub", "imul", "lmul", "fmul", "dmul", "idiv", "ldiv",
    "fdiv", "ddiv", "irem", "lrem", "frem", "drem", "ineg", "lneg",
    "fneg", "dneg", "ishl", "lshl", "ishr", "lshr", "iushr", "lushr",
    "iand", "land", "ior", "lor", "ixor", "lxor", "iinc", "i2l", "i2f",
    "i2d", "l2i", "l2f", "l2d", "f2i", "f2l", "f2d", "d2i", "d2l", "d2f",
    "i2b", "i2c", "i2s", "lcmp", "fcmpl", "fcmpg",
    "dcmpl", "dcmpg", "ifeq", "ifne", "iflt", "ifge", "ifgt", "ifle",
    "if_icmpeq", "if_icmpne", "if_icmplt", "if_icmpge", "if_icmpgt",
    "if_icmple", "if_acmpeq", "if_acmpne", "goto", "jsr", "ret",
    "tableswitch", "lookupswitch", "ireturn", "lreturn", "freturn",
    "dreturn", "areturn", "return", "getstatic", "putstatic", "getfield",
    "putfield", "invokevirtual", "invokespecial", "invokestatic",
    "invokeinterface", ILLEGAL_OPCODE, "new", "newarray", "anewarray",
    "arraylength", "athrow", "checkcast", "instanceof", "monitorenter",
    "monitorexit", "wide", "multianewarray", "ifnull", "ifnonnull",
    "goto_w", "jsr_w", "breakpoint", ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE, ILLEGAL_OPCODE,
    ILLEGAL_OPCODE, "impdep1", "impdep2"
  };

  /**
   * Number of words consumed on operand stack by instructions.
   */
  public static final int[] CONSUME_STACK = {
    0/*nop*/, 0/*aconst_null*/, 0/*iconst_m1*/, 0/*iconst_0*/, 0/*iconst_1*/,
    0/*iconst_2*/, 0/*iconst_3*/, 0/*iconst_4*/, 0/*iconst_5*/, 0/*lconst_0*/,
    0/*lconst_1*/, 0/*fconst_0*/, 0/*fconst_1*/, 0/*fconst_2*/, 0/*dconst_0*/,
    0/*dconst_1*/, 0/*bipush*/, 0/*sipush*/, 0/*ldc*/, 0/*ldc_w*/, 0/*ldc2_w*/, 0/*iload*/,
    0/*lload*/, 0/*fload*/, 0/*dload*/, 0/*aload*/, 0/*iload_0*/, 0/*iload_1*/, 0/*iload_2*/,
    0/*iload_3*/, 0/*lload_0*/, 0/*lload_1*/, 0/*lload_2*/, 0/*lload_3*/, 0/*fload_0*/,
    0/*fload_1*/, 0/*fload_2*/, 0/*fload_3*/, 0/*dload_0*/, 0/*dload_1*/, 0/*dload_2*/,
    0/*dload_3*/, 0/*aload_0*/, 0/*aload_1*/, 0/*aload_2*/, 0/*aload_3*/, 2/*iaload*/,
    2/*laload*/, 2/*faload*/, 2/*daload*/, 2/*aaload*/, 2/*baload*/, 2/*caload*/, 2/*saload*/,
    1/*istore*/, 2/*lstore*/, 1/*fstore*/, 2/*dstore*/, 1/*astore*/, 1/*istore_0*/,
    1/*istore_1*/, 1/*istore_2*/, 1/*istore_3*/, 2/*lstore_0*/, 2/*lstore_1*/,
    2/*lstore_2*/, 2/*lstore_3*/, 1/*fstore_0*/, 1/*fstore_1*/, 1/*fstore_2*/,
    1/*fstore_3*/, 2/*dstore_0*/, 2/*dstore_1*/, 2/*dstore_2*/, 2/*dstore_3*/,
    1/*astore_0*/, 1/*astore_1*/, 1/*astore_2*/, 1/*astore_3*/, 3/*iastore*/, 4/*lastore*/,
    3/*fastore*/, 4/*dastore*/, 3/*aastore*/, 3/*bastore*/, 3/*castore*/, 3/*sastore*/,
    1/*pop*/, 2/*pop2*/, 1/*dup*/, 2/*dup_x1*/, 3/*dup_x2*/, 2/*dup2*/, 3/*dup2_x1*/,
    4/*dup2_x2*/, 2/*swap*/, 2/*iadd*/, 4/*ladd*/, 2/*fadd*/, 4/*dadd*/, 2/*isub*/, 4/*lsub*/,
    2/*fsub*/, 4/*dsub*/, 2/*imul*/, 4/*lmul*/, 2/*fmul*/, 4/*dmul*/, 2/*idiv*/, 4/*ldiv*/,
    2/*fdiv*/, 4/*ddiv*/, 2/*irem*/, 4/*lrem*/, 2/*frem*/, 4/*drem*/, 1/*ineg*/, 2/*lneg*/,
    1/*fneg*/, 2/*dneg*/, 2/*ishl*/, 3/*lshl*/, 2/*ishr*/, 3/*lshr*/, 2/*iushr*/, 3/*lushr*/,
    2/*iand*/, 4/*land*/, 2/*ior*/, 4/*lor*/, 2/*ixor*/, 4/*lxor*/, 0/*iinc*/,
    1/*i2l*/, 1/*i2f*/, 1/*i2d*/, 2/*l2i*/, 2/*l2f*/, 2/*l2d*/, 1/*f2i*/, 1/*f2l*/,
    1/*f2d*/, 2/*d2i*/, 2/*d2l*/, 2/*d2f*/, 1/*i2b*/, 1/*i2c*/, 1/*i2s*/,
    4/*lcmp*/, 2/*fcmpl*/, 2/*fcmpg*/, 4/*dcmpl*/, 4/*dcmpg*/, 1/*ifeq*/, 1/*ifne*/,
    1/*iflt*/, 1/*ifge*/, 1/*ifgt*/, 1/*ifle*/, 2/*if_icmpeq*/, 2/*if_icmpne*/, 2/*if_icmplt*/,
    2 /*if_icmpge*/, 2/*if_icmpgt*/, 2/*if_icmple*/, 2/*if_acmpeq*/, 2/*if_acmpne*/,
    0/*goto*/, 0/*jsr*/, 0/*ret*/, 1/*tableswitch*/, 1/*lookupswitch*/, 1/*ireturn*/,
    2/*lreturn*/, 1/*freturn*/, 2/*dreturn*/, 1/*areturn*/, 0/*return*/, 0/*getstatic*/,
    UNPREDICTABLE/*putstatic*/, 1/*getfield*/, UNPREDICTABLE/*putfield*/,
    UNPREDICTABLE/*invokevirtual*/, UNPREDICTABLE/*invokespecial*/,
    UNPREDICTABLE/*invokestatic*/,
    UNPREDICTABLE/*invokeinterface*/, UNDEFINED, 0/*new*/, 1/*newarray*/, 1/*anewarray*/,
    1/*arraylength*/, 1/*athrow*/, 1/*checkcast*/, 1/*instanceof*/, 1/*monitorenter*/,
    1/*monitorexit*/, 0/*wide*/, UNPREDICTABLE/*multianewarray*/, 1/*ifnull*/, 1/*ifnonnull*/,
    0/*goto_w*/, 0/*jsr_w*/, 0/*breakpoint*/, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNPREDICTABLE/*impdep1*/, UNPREDICTABLE/*impdep2*/
  };

  /**
   * Number of words produced onto operand stack by instructions.
   */
  public static final int[] PRODUCE_STACK = {
    0/*nop*/, 1/*aconst_null*/, 1/*iconst_m1*/, 1/*iconst_0*/, 1/*iconst_1*/,
    1/*iconst_2*/, 1/*iconst_3*/, 1/*iconst_4*/, 1/*iconst_5*/, 2/*lconst_0*/,
    2/*lconst_1*/, 1/*fconst_0*/, 1/*fconst_1*/, 1/*fconst_2*/, 2/*dconst_0*/,
    2/*dconst_1*/, 1/*bipush*/, 1/*sipush*/, 1/*ldc*/, 1/*ldc_w*/, 2/*ldc2_w*/, 1/*iload*/,
    2/*lload*/, 1/*fload*/, 2/*dload*/, 1/*aload*/, 1/*iload_0*/, 1/*iload_1*/, 1/*iload_2*/,
    1/*iload_3*/, 2/*lload_0*/, 2/*lload_1*/, 2/*lload_2*/, 2/*lload_3*/, 1/*fload_0*/,
    1/*fload_1*/, 1/*fload_2*/, 1/*fload_3*/, 2/*dload_0*/, 2/*dload_1*/, 2/*dload_2*/,
    2/*dload_3*/, 1/*aload_0*/, 1/*aload_1*/, 1/*aload_2*/, 1/*aload_3*/, 1/*iaload*/,
    2/*laload*/, 1/*faload*/, 2/*daload*/, 1/*aaload*/, 1/*baload*/, 1/*caload*/, 1/*saload*/,
    0/*istore*/, 0/*lstore*/, 0/*fstore*/, 0/*dstore*/, 0/*astore*/, 0/*istore_0*/,
    0/*istore_1*/, 0/*istore_2*/, 0/*istore_3*/, 0/*lstore_0*/, 0/*lstore_1*/,
    0/*lstore_2*/, 0/*lstore_3*/, 0/*fstore_0*/, 0/*fstore_1*/, 0/*fstore_2*/,
    0/*fstore_3*/, 0/*dstore_0*/, 0/*dstore_1*/, 0/*dstore_2*/, 0/*dstore_3*/,
    0/*astore_0*/, 0/*astore_1*/, 0/*astore_2*/, 0/*astore_3*/, 0/*iastore*/, 0/*lastore*/,
    0/*fastore*/, 0/*dastore*/, 0/*aastore*/, 0/*bastore*/, 0/*castore*/, 0/*sastore*/,
    0/*pop*/, 0/*pop2*/, 2/*dup*/, 3/*dup_x1*/, 4/*dup_x2*/, 4/*dup2*/, 5/*dup2_x1*/,
    6/*dup2_x2*/, 2/*swap*/, 1/*iadd*/, 2/*ladd*/, 1/*fadd*/, 2/*dadd*/, 1/*isub*/, 2/*lsub*/,
    1/*fsub*/, 2/*dsub*/, 1/*imul*/, 2/*lmul*/, 1/*fmul*/, 2/*dmul*/, 1/*idiv*/, 2/*ldiv*/,
    1/*fdiv*/, 2/*ddiv*/, 1/*irem*/, 2/*lrem*/, 1/*frem*/, 2/*drem*/, 1/*ineg*/, 2/*lneg*/,
    1/*fneg*/, 2/*dneg*/, 1/*ishl*/, 2/*lshl*/, 1/*ishr*/, 2/*lshr*/, 1/*iushr*/, 2/*lushr*/,
    1/*iand*/, 2/*land*/, 1/*ior*/, 2/*lor*/, 1/*ixor*/, 2/*lxor*/,
    0/*iinc*/, 2/*i2l*/, 1/*i2f*/, 2/*i2d*/, 1/*l2i*/, 1/*l2f*/, 2/*l2d*/, 1/*f2i*/,
    2/*f2l*/, 2/*f2d*/, 1/*d2i*/, 2/*d2l*/, 1/*d2f*/,
    1/*i2b*/, 1/*i2c*/, 1/*i2s*/, 1/*lcmp*/, 1/*fcmpl*/, 1/*fcmpg*/,
    1/*dcmpl*/, 1/*dcmpg*/, 0/*ifeq*/, 0/*ifne*/, 0/*iflt*/, 0/*ifge*/, 0/*ifgt*/, 0/*ifle*/,
    0/*if_icmpeq*/, 0/*if_icmpne*/, 0/*if_icmplt*/, 0/*if_icmpge*/, 0/*if_icmpgt*/,
    0/*if_icmple*/, 0/*if_acmpeq*/, 0/*if_acmpne*/, 0/*goto*/, 1/*jsr*/, 0/*ret*/,
    0/*tableswitch*/, 0/*lookupswitch*/, 0/*ireturn*/, 0/*lreturn*/, 0/*freturn*/,
    0/*dreturn*/, 0/*areturn*/, 0/*return*/, UNPREDICTABLE/*getstatic*/, 0/*putstatic*/,
    UNPREDICTABLE/*getfield*/, 0/*putfield*/, UNPREDICTABLE/*invokevirtual*/,
    UNPREDICTABLE/*invokespecial*/, UNPREDICTABLE/*invokestatic*/,
    UNPREDICTABLE/*invokeinterface*/, UNDEFINED, 1/*new*/, 1/*newarray*/, 1/*anewarray*/,
    1/*arraylength*/, 1/*athrow*/, 1/*checkcast*/, 1/*instanceof*/, 0/*monitorenter*/,
    0/*monitorexit*/, 0/*wide*/, 1/*multianewarray*/, 0/*ifnull*/, 0/*ifnonnull*/,
    0/*goto_w*/, 1/*jsr_w*/, 0/*breakpoint*/, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNDEFINED, UNDEFINED, UNDEFINED,
    UNDEFINED, UNPREDICTABLE/*impdep1*/, UNPREDICTABLE/*impdep2*/
  };

  /** Attributes and their corresponding names.
   */
  public static final byte ATTR_UNKNOWN                                 = -1;
  public static final byte ATTR_SOURCE_FILE                             = 0;
  public static final byte ATTR_CONSTANT_VALUE                          = 1;
  public static final byte ATTR_CODE                                    = 2;
  public static final byte ATTR_EXCEPTIONS                              = 3;
  public static final byte ATTR_LINE_NUMBER_TABLE                       = 4;
  public static final byte ATTR_LOCAL_VARIABLE_TABLE                    = 5;
  public static final byte ATTR_INNER_CLASSES                           = 6;
  public static final byte ATTR_SYNTHETIC                               = 7;
  public static final byte ATTR_DEPRECATED                              = 8;
  public static final byte ATTR_PMG                                     = 9;
  public static final byte ATTR_SIGNATURE                               = 10;
  public static final byte ATTR_STACK_MAP                               = 11;
  public static final byte ATTR_LOCAL_VARIABLE_TYPE_TABLE               = 12;

  public static final short KNOWN_ATTRIBUTES = 13;

  public static final String[] ATTRIBUTE_NAMES = {
    "SourceFile", "ConstantValue", "Code", "Exceptions",
    "LineNumberTable", "LocalVariableTable",
    "InnerClasses", "Synthetic", "Deprecated",
    "PMGClass", "Signature", "StackMap",
    "LocalVariableTypeTable"
  };

  /** Constants used in the StackMap attribute.
   */
  public static final byte ITEM_Bogus      = 0;
  public static final byte ITEM_Integer    = 1;
  public static final byte ITEM_Float      = 2;
  public static final byte ITEM_Double     = 3;
  public static final byte ITEM_Long       = 4;
  public static final byte ITEM_Null       = 5;
  public static final byte ITEM_InitObject = 6;
  public static final byte ITEM_Object     = 7;
  public static final byte ITEM_NewObject  = 8;

  public static final String[] ITEM_NAMES = {
    "Bogus", "Integer", "Float", "Double", "Long",
    "Null", "InitObject", "Object", "NewObject"
  };
}


```