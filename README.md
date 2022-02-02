# cocproxy
Clash of Clans Proxy with some boot options

TranslationsList.js
import React, {Component} from 'react';
import {
    CustomPaging,
    DataTypeProvider,
    EditingState,
    IntegratedSorting,
    PagingState,
    SortingState
} from '@devexpress/dx-react-grid';
import {
    Grid,
    PagingPanel,
    DragDropProvider,
    TableColumnReordering,
    TableColumnResizing,
    TableEditColumn,
    TableEditRow,
    TableHeaderRow,
    VirtualTable,
    Table
} from '@devexpress/dx-react-grid-bootstrap4';
import '@devexpress/dx-react-grid-bootstrap4/dist/dx-react-grid-bootstrap4.css';
import {connect} from "react-redux";
import {translationActions} from "../../_actions";
import {
    Button,
    ButtonDropdown,
    Card,
    CardBody,
    CardHeader,
    Col,
    Container,
    CustomInput,
    Dropdown,
    DropdownItem,
    DropdownMenu,
    DropdownToggle,
    Input,
    InputGroup,
    InputGroupAddon,
    Row, UncontrolledTooltip
} from "reactstrap";

import {FaSearch} from "react-icons/fa";
import {trackPromise} from "react-promise-tracker";
import {translationService} from "../../_services/translations.service";
import TooltipFormatter from "./TooltipFormatter";
import confirm from "reactstrap-confirm";
import {LoadingIndicator} from "../LoadingIndicator";
import GridRootComponent from "./GridRootComponent";
import Nope from "nope-validator";
import Select from "react-select";
import {config} from "../../_config/Config";

const validationSchema = Nope.object().shape({
    component: Nope.string().required("component is required").greaterThan(1),
    key: Nope.string().required("text key is required").greaterThan(1),
    language: Nope.string().required("language is required").greaterThan(1),
    channel: Nope.string().required("channel is required").greaterThan(1)
});


const PAGE_SIZE = 20;
const TLIST_AREA = "tlist-area"; // area where the loading indicator will be shown
const COPY_ID_PREFIX = "copy:"; // during copy of row the temporary id prefix of the copied row;  if copy row with id = 101 then the copied row has id copy:101
const DEFAULT_SORT_EXPORT = [
    {columnName: "component", direction: "asc"},
    {columnName: "key", direction: "asc"},
    {columnName: "language", direction: "asc"},
    {columnName: "channel", direction: "asc"}
];
const defaultColumnWidths = [
    { columnName: 'component', width: 160 },
    { columnName: 'key', width: 400 },
    { columnName: 'language', width: 130 },
    { columnName: 'channel', width: 130 },
    { columnName: 'text', width: 640 },
    { columnName: 'description', width: 300 },
    ];

const columns = [
    {name: 'component', title: 'Component'},
    {name: 'key', title: 'Key'},
    {name: 'language', title: 'Language'},
    {name: 'channel', title: 'Channel'},
    {name: 'text', title: 'Text'},
    {name: 'description', title: 'Description'}
];

const columnNames = columns.map(column => column.name);

const columnExtensions = [
    { columnName: 'text', wordWrapEnabled: true },
    { columnName: 'description', wordWrapEnabled: true }
];

const sortingColumnExtensions = [
    { columnName: 'component', sortingEnabled: true },
    { columnName: 'key', sortingEnabled: true },
    { columnName: 'language', sortingEnabled: true },
    { columnName: 'channel', sortingEnabled: true },
    { columnName: 'text', sortingEnabled: true },
    { columnName: 'description', sortingEnabled: true }
];

const editingStateColumnExtensions = [
    {columnName: 'component', editingEnabled: true},
    {columnName: 'language', editingEnabled: true},
    {columnName: 'channel', editingEnabled: true}
];

/* grid cell editors and providers */
// const BooleanEditor = ({ value, onValueChange }) => (
//   <select
//     className="form-control"
//     value={value}
//     onChange={e => onValueChange(e.target.value === 'true')}
//   >
//       <option value={false}>
//           No
//       </option>
//       <option value>
//           Yes
//       </option>
//   </select>
// );
//
// const BooleanTypeProvider = props => (
//   <DataTypeProvider
//     editorComponent={BooleanEditor}
//     {...props}
//   />
// );
const EditCell = ({ errors, ...props }) => {
    let children = props.children;
    const currentRow = props.tableRow.row;
    const rowErrors = errors?.[currentRow.id];
    if (rowErrors != null) {
        children =
          React.Children.map(
            props.children
            , child => child && child.props.id === 'commit' ? React.cloneElement(child, { disabled: true}) : child
          );
    }
    return (
      <TableEditColumn.Cell {...props}>
          {children}
          {props.onCopy && <CopyButton onExecute={() => props.onCopy(currentRow)} />}
      </TableEditColumn.Cell>
    );
};

const InputEditor = ({value, onValueChange, column, ...rest}) => {
    return (
      <Input name={column.name}
             value={value || ''}
             onBlur={e => onValueChange(e.target.value.trim())}
             onChange={e => onValueChange(e.target.value)}
             className="w-100"/>)
};

const InputTypeProvider = (props) => (
  <DataTypeProvider
    editorComponent={InputEditor}
    {...props}
  />
);

const TextEditor = ({value, onValueChange, column}) => (
  <Input type="textarea"
         name={column.name}
         value={value || ''}
         onChange={e => onValueChange(e.target.value)}
         className="w-100"/>
);

const TextTypeProvider = (props) => (
  <DataTypeProvider
    editorComponent={TextEditor}
    {...props}
  />
);

const OptionsEditorFactory = (options) => {
    return ({value, onValueChange, column, ...rest}) => {
        const selOpt = options.find(c => c.value === value);
        return (
          <Select options={options}
                  value={selOpt}
                  onChange={opt => onValueChange(opt.value)}
                  className="w-100"/>
        );
    }
}

const OptionsTypeProvider = ({options, ...props}) => {
    return (
      <DataTypeProvider
        editorComponent={React.useMemo(() => OptionsEditorFactory(options), [options])}
        // editorComponent={ComponentEditorFactory(components)}
        {...props}
      />
    )
};

const CommandButton = ({onExecute, icon, text, hint, color, disabled}) => (
  <button
    type="button"
    className="btn btn-link"
    style={{ padding: 11, cursor: disabled ? "not-allowed" : null, "pointerEvents": disabled ? "all": null }}
    onClick={(e) => {
        onExecute();
        e.stopPropagation();
    }}
    title={hint}
    disabled={disabled}
  >
    <span className={color || 'undefined'}>
      {icon ? <i className={`oi oi-${icon}`} style={{ marginRight: text ? 5 : 0}} /> : null}
        {text}
    </span>
  </button>
);

const AddButton = ({ onExecute }) => (
  <CommandButton icon="plus" hint="Create new translation" onExecute={onExecute} />
);

const CopyButton = ({ onExecute }) => (
  <CommandButton icon="plus" hint="Copy translation" color="text-success" onExecute={onExecute} />
);

const EditButton = ({ onExecute }) => (
  <CommandButton icon="pencil" hint="Edit translation" color="text-warning" onExecute={onExecute} />
);

const DeleteButton = ({ onExecute }) => (
  <CommandButton
    icon="trash"
    hint="Delete translation"
    color="text-danger"
    onExecute={async () => {
        // eslint-disable-next-line
        if (await confirm({message: 'Are you sure you want to delete this translation ?'})) {
            onExecute();
        }
    }}
  />
);

const CommitButton = ({ onExecute, ...rest }) => {
    return (
      <CommandButton icon="check" hint="Save changes" color="text-success" onExecute={onExecute} {...rest}/>
    )
};

const CancelButton = ({ onExecute }) => (
  <CommandButton icon="x" hint="Cancel changes" color="text-danger" onExecute={onExecute} />
);

const commandComponents = {
    add: AddButton,
    edit: EditButton,
    delete: DeleteButton,
    commit: CommitButton,
    cancel: CancelButton,
};

const Command = ({ id, onExecute, disabled }) => {
    const ButtonComponent = commandComponents[id];
    return (
      <ButtonComponent
        onExecute={onExecute}
        disabled={disabled}
      />
    );
};

class CellTooltip extends Component {
    render() {
        return (
          <DataTypeProvider
            for={columnNames}
            formatterComponent={(props) => <TooltipFormatter {...props} {...this.props}/>}
            {...this.props}
          />
        )
    }
};

/* grid helpers */
// alternative to index-id //const getRowId = row => "{\"tenant\":\"" + row.tenant + "\",\"component\":\"" + row.component + "\",\"key\":\"" + row.key + "\",\"language\":\"" + row.language + "\"}";

class TranslationsListComponent extends Component {

    constructor(props) {
        super(props);

        this.state = {
            selectedFile:'',
            fileInputKey: Date.now(),
            searchBy: this.props.searchBy,
            highlightBy: '',
            highlightByRegExp: null,
            pager: null,
            pageableQuery: null, //(page, size) => {}
            exportDropDownOpen: false,
            sorts: null,
            editingErrors: null,
            addedRows: [],
            editingRowIds: []
        }
    }

    static getDerivedStateFromProps(nextProps, prevState){ // TODO: keep tenant in prop only ???
        if(nextProps.searchBy !== prevState.searchBy){
            return {
                searchBy: nextProps.searchBy,
                highlightBy: nextProps.searchBy,
                highlightByRegExp: TranslationsListComponent.toRegExp(nextProps.searchBy)
            };
        }
        else return null;
    }

    componentDidMount() {
        if (this.props.searchBy && this.props.searchBy?.trim()?.length !== 0) {
            this.searchByText(this.props.searchBy);
        } else {
            this.searchByParams(this.props.tenant, this.props.component, this.props.language, this.props.channel, this.props.textKey, this.props.text, this.props.description);
        }
    }

    componentDidUpdate = (prevProps, prevState) => {
        if(prevProps.tenant !== this.props.tenant
          || prevProps.searchBy !== this.props.searchBy
          || prevProps.component !== this.props.component
          || prevProps.language !== this.props.language
          || prevProps.channel !== this.props.channel
          || prevProps.textKey !== this.props.textKey
          || prevProps.text !== this.props.text
          || prevProps.description !== this.props.description
          || prevProps.tick !== this.props.tick
        ){
            //Perform some operation here
            if (this.props.searchBy && this.props.searchBy.trim()?.length !== 0) {
                this.searchByText(this.props.searchBy);
            } else {
                this.searchByParams(this.props.tenant, this.props.component, this.props.language, this.props.channel, this.props.textKey, this.props.text, this.props.description);
            }
        }
    }

    searchByText  = (searchBy) => {
        console.info(`searchByText '${searchBy}'`);
        if (!searchBy) {
            console.warn("no searchBy");
            return;
        }
        searchBy = searchBy.trim();
        if (0 === searchBy.length) {
            console.warn("empty searchBy");
            this.setState({
                searchBy,
            })
            return ;
        }

        const pageableQuery = (page, size, sorts, exportOptions) => (
          trackPromise(
            translationService.search(page, size, this.props.tenant, searchBy, sorts, exportOptions).then(
              pager => {
                  if (!exportOptions) {
                      this.setState({pager, sorts})
                  }
                  return pager;
              })
            , TLIST_AREA)
        );

        this.setState({
            pageableQuery: pageableQuery,
            searchBy: searchBy
        });

        return pageableQuery(0, PAGE_SIZE)
          .catch(error => {
              this.confirmError(`Failed search translation. (${error})`);
          });
    }

    /* safe construction of RegExp from pattern */
    static toRegExp(pattern) {
        try {
            return pattern ? new RegExp(pattern, 'gi') : null
        } catch (e) {
            console.warn(`invalid regexp: '${pattern}'`);
        }
    }

    handleHighlightOnChange = (e) => {
        const highlightBy = e.target.value;
        const highlightByRegExp = TranslationsListComponent.toRegExp(highlightBy)
        this.setState({
            highlightBy,
            highlightByRegExp
        })
    }

    // handleKeyPress = (e) => {
    //     if (e.charCode !== 13 || !e.target.value) return;
    //     e.preventDefault();
    //     if (!this.props.tenant) {
    //         console.warn("no tenant");
    //         return;
    //     }
    //     const searchBy = e.target.value.trim();
    //     if (0 === searchBy.length) {
    //         this.setState({
    //             searchBy,
    //             searchByRegExp: null
    //         })
    //         return ;
    //     }
    //
    //     const pageableQuery = (page, size, sorts, exportOptions) => (
    //       trackPromise(
    //         translationService.search(page, size, this.props.tenant, searchBy, sorts, exportOptions).then(
    //           pager => {
    //               if (!exportOptions) {
    //                   this.setState({pager, sorts})
    //               }
    //               return pager;
    //           })
    //         , TLIST_AREA)
    //     );
    //
    //     this.setState({
    //         pageableQuery: pageableQuery,
    //         searchBy,
    //         searchByRegExp: new RegExp(searchBy, 'gi')
    //     });
    //
    //     return pageableQuery(0, PAGE_SIZE)
    //       .catch(error => {
    //           this.confirmError(`Failed search translation. (${error})`);
    //       });
    // }

    handleImport = () => {
        if (!this.state.selectedFile) {
            console.warn("No file selected for import");
            return;
        }

        trackPromise(
          translationService.import(this.state.selectedFile, this.props.tenant).then(
            result => {
                if (!result) {
                    this.notify("Translations imported. ");
                } else { // some duplicates
                    this.notify("Translations imported. Check returned 'rejections...' file for rejections.");
                }
                this.setState({
                    selectedFile: '',
                    fileInputKey: Date.now()
                }, this.reload);
          }).catch(error => {
              this.confirmError(`Failed import translations. (${error})`);
          })
          ,TLIST_AREA
        )
    }

    fileSelectedHandler = (event) => {
        this.setState({selectedFile: event.target.files[0]});
    }

    searchByParams = (tenant, component, language, channel, key, text, description, page = 0, size = PAGE_SIZE) => {
        console.info("searchByParams", tenant, component, language, channel, key, text, description, page, size);
        if (!tenant) {
            console.error("No tenant provided for loading translations");
            return;
        }
        const pageableQuery = (page, size, sorts, exportOptions) => (
          trackPromise(
              translationService.searchPaginatedByParams(page, size, tenant, component, language, channel, key, text, description, sorts, exportOptions).then(
                pager => {
                    if (!exportOptions) {
                        this.setState({pager, sorts})
                    }
                    return pager;
                })
            ,TLIST_AREA)
        );

        this.setState({
            pageableQuery: pageableQuery
        });

        return pageableQuery(page, size)
          .catch(error => {this.confirmError(`Failed search translations. (${error})`);})

    }

    toggleExportDropDown = () => {
        this.setState({
            exportDropDownOpen: !this.state.exportDropDownOpen,
        })
    }

    export = (asXls, sorts) => {
        if (typeof this.state.pageableQuery === "function") {
            this.state.pageableQuery(null, null, sorts || this.state.sorts, {asXls})
              .then(response => this.notify(response ? "Translations exported. Check downloaded file..." : "No translations found."))
              .catch(error => this.confirmError(`Failed export (${error})`));
        }
    }

    confirmError = async (message) => {
        await confirm({
            title: 'Error!',
            message,
            cancelText: null,
            confirmText: 'OK'
        });
    }

    notify = async (message) => {
        await confirm({
            title: 'Success!',
            message,
            cancelText: null,
            confirmText: 'OK'
        })
    }

    setCurrentPage = (page = 0, size = PAGE_SIZE) => {
        if (typeof this.state.pageableQuery === "function") {
            this.state.pageableQuery(page, size, this.state.sorts)
              .catch(error => {this.confirmError(`Failed search translations. (${error})`);})
        }
    }

    reload = () => {
        // reload tenants and components
        this.props.getTenants();
        if (typeof this.state.pageableQuery === "function") {
            const page = this.state.pager.number || 0;
            const size = this.state.pager.size || PAGE_SIZE;
            const sorts = this.state.sorts;
            this.state.pageableQuery(page, size, sorts)
              .catch(error => {this.confirmError(`Failed reload translations. (${error})`);})
        }
    }

    onEditingRowIds = async (editingRowIds) => {
        this.setState({editingRowIds});
    }
    
    onEdited = async (edited, rows) => {
        const errors =
          Object.entries(edited).map(([index, changes]) => {
              const editedRow = {...rows[index], ...changes};
              const err = validationSchema.validate(editedRow);
              return err && {id: editedRow.id, err};
          }).reduce((accumulator, value) => {
              if (value) {
                  const {id, err} = value;
                  accumulator[id] = err;
              }
              return accumulator;
          }, {});
        this.setState({editingErrors: errors});
    }

    onAdded = async (added, rows) => {
        const errors =
          Object.entries(added).map(([index, changes]) => {
              const editedRow = {...rows[index], ...changes};
              const err = validationSchema.validate(editedRow);
              return err && {id: editedRow.id, err};
          }).reduce((accumulator, value) => {
              if (value) {
                  const {id, err} = value;
                  accumulator[id] = err;
              }
              return accumulator;
          }, {});
        this.setState({addedRows: added, editingErrors: errors});
    }

    onCopy = async (row) => {
        const addedRows = this.state.addedRows.slice();
        const copyRow = {...row, id: COPY_ID_PREFIX + row.id};
        addedRows.push(copyRow);
        this.setState({addedRows});
    }

    commitChanges = ({changed, deleted, added}, rows) => {
        if (added) { // TODO multiple adds
            const addedRows = Object.entries(added).map(([index, adds]) => ({...rows[index], ...adds}));
            console.log("rows to add: " + JSON.stringify(addedRows));
            addedRows.map(
              addedRow =>
                trackPromise(
                  translationService.create(addedRow.tenant, addedRow.component, addedRow.language, addedRow.channel, addedRow.key, addedRow.text, addedRow.description).then(value => {
                      this.notify("Translation created.");
                      // reload grid
                      this.setCurrentPage(this.state.pager?.number, this.state.pager?.size)
                  }).catch(error => {
                      this.confirmError(`Failed translation create. (error: ${error})`);
                  })
                  , TLIST_AREA)
            )
        }
        if (changed) { // TODO multiple updates
            const changedRows = Object.entries(changed).map(([index, changes]) => ({...rows[index], ...changes}));
            console.log("rows to update: " + JSON.stringify(changedRows));
            changedRows.map(
              changedRow =>
                  trackPromise(
                    translationService.update(changedRow).then(value => {
                        this.notify("Translation updated.");
                        // reload grid
                        this.setCurrentPage(this.state.pager?.number, this.state.pager?.size)
                    }).catch(error => {
                        this.confirmError(`Failed translation update. (error: ${error})`);
                    })
                    , TLIST_AREA)
            )
        }
        if (deleted) {
            // find deleted row // TODO multiple deletes
            const deletedRows = deleted.map(index => rows[index]);
            console.log("rows to delete: " + JSON.stringify(deletedRows));
            deletedRows.map(
              row => {
                  return trackPromise(
                    translationService.delete(row).then(value => {
                        this.notify("Translation deleted.");
                        // reload grid
                        this.setCurrentPage(this.state.pager?.number, this.state.pager?.size)
                    }).catch(error => {
                        this.confirmError(`Failed translation delete. (error: ${error})`);
                    })
                    , TLIST_AREA)
              }
            )
        }
    };

    setSorting = (sorts) => {
        this.setState({
            sorts
        }, this.reload);
    }

    tableRow = ({row, ...restProps}) => { // visually mark the current copying rows
        const style = (this.state.addedRows?.find(r => r.id === row.id || (r.id.startsWith(COPY_ID_PREFIX) && r.id === COPY_ID_PREFIX + row.id)) && {backgroundColor: '#e9ecef'}) || null;
        return (
          <Table.Row {...restProps} style={style}/>
        )
    }

    render() {

        const rows = this.state.pager?.content || [];
        const sorting = this.state.sorts || [];
        const addedRows = this.state.addedRows || [];
        const editingRowIds = this.state.editingRowIds || [];
        return (
            <Card className="h-100 pr-0">
                <CardHeader>
                    <Container fluid={true}>
                        <Row>
                            <Col className="col-1">
                                <Button color="secondary"
                                        onClick={this.reload}
                                        disabled={!this.state.pageableQuery}>
                                    <span className="oi oi-reload"></span>
                                </Button>
                            </Col>
                            <Col className="col-3">
                                <CustomInput type="file" id="customFile" name="customFile" label="Choose a file..." onChange={this.fileSelectedHandler} key={this.state.fileInputKey} />
                                {/*<Input type="file" onChange={this.fileSelectedHandler} key={this.state.fileInputKey} />*/}
                            </Col>
                            <Col className="col-1">
                                <Button color="primary" disabled={!this.state.selectedFile} onClick={this.handleImport}>Import</Button>
                            </Col>
                            <Col className="col-2"></Col>
                            <Col className="col-1">
                                <ButtonDropdown>
                                    <Dropdown isOpen={this.state.exportDropDownOpen} toggle={this.toggleExportDropDown}>
                                        <DropdownToggle caret disabled={!this.state.pager?.totalElements}>
                                            Export
                                        </DropdownToggle>
                                        <DropdownMenu>
                                            <DropdownItem onClick={() => this.export(true)}>as Excel</DropdownItem>
                                            <DropdownItem onClick={() => this.export(false)}>as JSON</DropdownItem>
                                            <DropdownItem onClick={() => this.export(false, DEFAULT_SORT_EXPORT)}>as JSON sorted by (component, key, language, channel)</DropdownItem>
                                        </DropdownMenu>
                                    </Dropdown>
                                </ButtonDropdown>
                            </Col>
                            <Col className="col-4">
                                <InputGroup id="highlightInput">
                                    <Input name="highlightBy"
                                           value={this.state.highlightBy || ''}
                                           onChange={this.handleHighlightOnChange}
                                           disabled={!this.props.tenant}
                                           style={{color: this.state.highlightByRegExp ? null : 'red'}}
                                           />
                                    <InputGroupAddon addonType="append">
                                        {/*<span className="input-group-text"><FaSearch/></span>*/}
                                        <a className="input-group-text" href="https://images.app.goo.gl/78pJpBHb2xzy5Gzr6" target="_blank" rel="noopener noreferrer">...<FaSearch/></a>
                                    </InputGroupAddon>
                                </InputGroup>
                                <UncontrolledTooltip
                                  placement="top"
                                  autohide={false}
                                  target="highlightInput"
                                  style={{backgroundColor: 'lightgrey'}}
                                >
                                    <a href="https://images.app.goo.gl/78pJpBHb2xzy5Gzr6" target="_blank" rel="noopener noreferrer">Regular Expressions Help</a>
                                </UncontrolledTooltip>
                            </Col>
                        </Row>
                    </Container>
                </CardHeader>
                <CardBody>
                    <LoadingIndicator area="tlist-area"/>
                    <Grid rows={rows}
                          columns={columns}
                          rootComponent={GridRootComponent}>
                        <SortingState
                          columnSortingEnabled={false}
                          columnExtensions={sortingColumnExtensions}
                          sorting={sorting}
                          onSortingChange={this.setSorting}
                        />
                        <DragDropProvider />
                        <OptionsTypeProvider for={['component']} options={this.props.components}/>
                        <OptionsTypeProvider for={['language']} options={this.props.languages}/>
                        <OptionsTypeProvider for={['channel']} options={this.props.channels}/>
                        <TextTypeProvider for={['text', 'description']}/>
                        <InputTypeProvider for={['key']}/>
                        <EditingState
                          editingRowIds={editingRowIds}
                          onEditingRowIdsChange={this.onEditingRowIds}
                          onCommitChanges={(changes) => this.commitChanges(changes, rows)}
                          onRowChangesChange={(edits) => this.onEdited(edits, rows)}
                          onAddedRowsChange={(adds) => this.onAdded(adds, rows)}
                          addedRows={addedRows}
                          columnExtensions={editingStateColumnExtensions}
                        />
                        {this.state.pager &&
                        <PagingState
                          currentPage={this.state.pager.number || 0}
                          onCurrentPageChange={this.setCurrentPage}
                          pageSize={this.state.pager.size || PAGE_SIZE}
                        />
                        }
                        {this.state.pager &&
                        <CustomPaging
                          totalCount={this.state.pager.totalElements || 0}
                        />
                        }
                        <CellTooltip highlightByRegExp={this.state.highlightByRegExp}/>
                        <IntegratedSorting/>
                        <VirtualTable
                          columnExtensions={columnExtensions}
                          rowComponent={this.tableRow}
                        />
                        {/*<Table columnExtensions={columnExtensions} />*/}
                        <TableColumnReordering defaultOrder={columnNames} />
                        <TableColumnResizing defaultColumnWidths={defaultColumnWidths} />
                        <TableHeaderRow showSortingControls/>
                        {this.state.pager &&
                        <PagingPanel />
                        }
                        <TableEditRow
                          rowComponent={this.tableRow}
                        />
                        <TableEditColumn
                          width={110}
                          showAddCommand={false}
                          showEditCommand
                          showDeleteCommand
                          commandComponent={Command}
                          cellComponent={props => <EditCell {...props} errors={this.state.editingErrors} onCopy={!this.state.addedRows.length && !this.state.editingRowIds.length ? this.onCopy : null} />}
                        />
                    </Grid>
                </CardBody>
            </Card>
        );
    }
}

const actionCreators = {
    delete: translationActions.delete,
    update: translationActions.update,
    search: translationActions.search,
    import: translationActions.import,
    getTenants: translationActions.getTenants
};

function mapState(state) {
    const {tick, items, searchBy, tenants, component, language, channel, textKey, text, description} = state.translations;
    const {tenant} = state.authentication;
    // after import the tenants components are updated
    const components = tenants?.find(t => t.value === tenant?.value)?.components || tenant?.components;
    components && components.sort();
    return {tick, items, tenant: tenant?.value, searchBy, component, language, channel, textKey, text, description, components: components?.map(c => ({value:c, label: c})), languages: tenant?.languages || config.languages, channels: tenant?.channels || config.channels}
}


const TranslationsList = connect(mapState, actionCreators)(TranslationsListComponent);
export default TranslationsList;

                  
translations.services.js
import {config} from '../_config/Config';
import * as authService from './auth.service'
//import {FaBloggerB} from "react-icons/all";
import {getAuthHeader} from '../_helpers';

export const translationService = {
    getTenants,
    create,
    searchPaginatedByParams,
    search,
    update,
    delete: _delete,
    import: _import,
};

function create(tenant, component, language, channel, key, text, description) {
    const url = config.url.API_URL;
    const requestOptions = {
        method: 'POST',
        headers: {...getAuthHeader(), 'Content-Type': 'application/json'},
        body: JSON.stringify({
            tenant: tenant,
            component: component,
            language: language,
            channel: channel,
            key: key,
            text: text,
            description: description
        })
    };
    return fetch(url, requestOptions).then(handleStatus).then(handleJson);
}

function update(translation) {
    const url = config.url.API_URL;
    const requestOptions = {
        method: 'PUT',
        headers: {...getAuthHeader(), 'Content-Type': 'application/json'},
        body: JSON.stringify(translation)
    };

    return fetch(url, requestOptions).then(handleStatus).then(handleJson);
}

function searchPaginatedByParams(page, size, tenant, component, language, channel, key, text, description, sorts, exportOptions) {
    let url = new URL(config.url.API_URL + "/tenant/" + tenant, window.location.origin);
    url.searchParams.append("lenient", true); // use 'contains' instead of 'equals" for comparisons.
    sorts && sorts.forEach(({columnName, direction}) => url.searchParams.append("sort", `${columnName},${direction}`));
    page != null && url.searchParams.append("page", page);
    size && url.searchParams.append("size", size);
    component && (Array.isArray(component) ? component.forEach(c => url.searchParams.append("component", c)): url.searchParams.append("component", component));
    language && url.searchParams.append("language", language);
    channel && (Array.isArray(channel) ? channel.forEach(c => url.searchParams.append("channel", c)) : url.searchParams.append("channel", channel));
    key && url.searchParams.append("key", key);
    text && url.searchParams.append("text", text);
    description && url.searchParams.append("description", description);

    const acceptType = exportOptions?.asXls ? "application/vnd.ms-excel" : "application/json";
    const requestOptions = {
        method: 'GET',
        headers: {
            ...getAuthHeader(),
            'Accept': acceptType,
            'Content-Type': 'application/json'
        }
    };

    const promise = fetch(url, requestOptions);
    return exportOptions ?
      promise.then(handleStatus).then(handleBlob).then(blob => download(blob, exportOptions.asXls ? "translations.xlsx" : "translations.json"))
      : promise.then(handleStatus).then(handleJson);
}

// export as json or xls
// function _export(tenant, component, language, key, text, description, asJson = true) {
//     if (tenant == null) {
//         return Promise.reject("Missing tenant");
//     }
//
//     let url = new URL(config.url.API_URL + "/export/tenant/" + tenant, window.location.origin);
//     url.searchParams.append("lenient", true); // use 'contains' instead of 'equals" for comparisons.
//     component && url.searchParams.append("component", component);
//     language && url.searchParams.append("language", language);
//     key && url.searchParams.append("key", key);
//     text && url.searchParams.append("text", text);
//     description && url.searchParams.append("description", description);
//
//     const acceptType = asJson ? "application/json" : "application/vnd.ms-excel";
//     const requestOptions = {
//         method: 'GET',
//         headers: {...getAuthHeader(), 'Accept': acceptType}
//     };
//     return fetch(url, requestOptions)
//       .then(response => response.blob()).then(blob => download(blob, asJson ? "translations.json" : "translations.xlsx"));
// }

function search(page, size, tenant, searchBy, sorts, exportOptions) {
    if (tenant == null) {
        return Promise.reject("Missing tenant");
    }
    if (searchBy == null) {
        return Promise.reject("Missing searchBy");
    }
    let url = new URL(config.url.API_URL + "/tenant/" + tenant + "/search", window.location.origin);
    url.searchParams.append("searchBy", searchBy);
    sorts && sorts.forEach(({columnName, direction}) => url.searchParams.append("sort", `${columnName},${direction}`));
    page != null && url.searchParams.append("page", page);
    size && url.searchParams.append("size", size);

    const acceptType = exportOptions?.asXls ? "application/vnd.ms-excel" : "application/json";
    const requestOptions = {
        method: 'GET',
        headers: {
            ...getAuthHeader(),
            'Accept': acceptType,
            'Content-Type': 'application/json'
        }
    };

    const promise = fetch(url, requestOptions);
    return exportOptions ?
      promise.then(handleStatus).then(handleBlob).then(blob => download(blob, exportOptions.asXls ? "translations.xlsx" : "translations.json"))
      : promise.then(handleStatus).then(handleJson);
}

function _delete({id}) {
    if (id == null) {
        return Promise.reject("Missing id for delete");
    }

    const url = config.url.API_URL + "/" + id;
    const requestOptions = {
        method: 'DELETE',
        headers: getAuthHeader()
    };

    return fetch(url, requestOptions).then(handleStatus).then(handleJson);
}

function getTenants() {
    const url = config.url.API_URL + "/noauth/tenants"
    const requestOptions = {
        method: 'POST',
        headers: getAuthHeader()
    };
    return fetch(url, requestOptions).then(handleStatus).then(handleJson)
      .then(json => {
          json.forEach(tenant => { // enhance with languages if missing
              if (tenant.languages == null || !tenant.languages.length) {
                  tenant.languages = config.languages;
              }
          });
          localStorage.setItem("tenants", JSON.stringify(json));
          return json;
      });
}

// function getById(id) {
//     const requestOptions = {
//         method: 'GET',
//         headers: getAuthHeader()
//     };
//
//     return fetch(`${config.apiUrl}/translations/${id}`, requestOptions).then(handleStatus).then(handleJson);
// }


function download(blob, fileName) {
    if (blob?.size) {
        // const blobContent = base64StringToBlob(jsonResult.translations, "text/plain");
        // const blobContent = new Blob([blob], {type: "application/vnd.ms-excel"})
        // works for PDF only ???
        // const file = new Blob([blob], {type: "application/vnd.ms-excel"});
        // const fileUrl = URL.createObjectURL(file);
        // if (window.navigator && window.navigator.msSaveOrOpenBlob) {
        //     window.navigator.msSaveOrOpenBlob(file, fileName);
        // } else {
        //     window.open(fileUrl);
        // }
        if (window.navigator && window.navigator.msSaveOrOpenBlob) {
            window.navigator.msSaveOrOpenBlob(blob, fileName);
        } else {
            const link = document.createElement('a');
            const blobUrl = URL.createObjectURL(blob);
            link.href = blobUrl;
            link.download = fileName;
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
        }
        return blob.size;
    } else {
        console.log("Nothing to download as ", fileName);
    }
}


function _import(selectedFile, tenant) {
    if (selectedFile == null) {
        return Promise.reject("Missing file");
    }

    if (tenant == null) {
        return Promise.reject("Missing tenant");
    }

    const url = config.url.API_URL + "/import"

    const formData = new FormData();
    formData.append("file", selectedFile);
    const requestOptions = {
        method: 'POST',
        headers: {
            ...getAuthHeader(),
            'v-tenant': tenant
        },
        body: formData
    };

    return fetch(url, requestOptions)
      .then(handleStatus)
      .then(handleBlob)
      .then(blob => download(blob,  "rejections.xlsx"));
}

async function handleStatus(response) {
    if (!response.ok) {
        if (response.status === 401) {
            authService.authService.logout(true);
        }
        const text = await response.text();
        const data = text && JSON.parse(text);
        const error = new Error ((data && data.message) || response.statusText);
        error.status = response.status;
        return Promise.reject(error);
    }
    return response;
}

async function handleBlob(response) {
    return response.blob();
}

async function handleJson(response) {
    return response.text().then(text => (text && JSON.parse(text)));
}

package eu.unicredit.dbn.translations.repositories;

import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;

import org.hibernate.Session;
import org.springframework.transaction.annotation.Transactional;

public class ClientIdentifierRepositoryCustomImpl
	implements ClientIdentifierRepositoryCustom {

	@PersistenceContext
	EntityManager entityManager;

	@Transactional
	@Override
	public void setClientIdentifier(String clientIdentifier) {
		Session session = entityManager.unwrap(Session.class);
		session.doWork(connection -> connection.setClientInfo("OCSID.CLIENTID", clientIdentifier));
	}
}
                    
create or replace trigger trg_audit_translations
    after update or delete or insert
    on TRANSLATIONS
    for each row
declare
    op varchar2(1) := null;
    userid varchar2(255) := null;
begin
    if (inserting or updating) then
        op := 'C';
        if updating then
            op := 'U';
        end if;
        insert into TRANSLATIONS_AUDIT
            (id, tenant, component, key, language, text, description, modified_by, modified_date, op)
            values
            (:NEW.id, :NEW.tenant, :NEW.component, :NEW.key, :NEW.language, :NEW.text, :NEW.description, :NEW.modified_by, :NEW.modified_date, op);
    elsif deleting then
        op := 'D';
        userid := nvl(sys_context('userenv', 'client_identifier'), user);
        insert into TRANSLATIONS_AUDIT
            (id, tenant, component, key, language, text, description, modified_by, modified_date, op)
            values
            (:OLD.id, :OLD.tenant, :OLD.component, :OLD.key, :OLD.language, :OLD.text, :OLD.description, userid, systimestamp, op);
    end if;
end;


